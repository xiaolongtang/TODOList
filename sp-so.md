思路可行，而且只用 Spring Boot 自带能力就能把它做得很稳。下面给你一套**零三方依赖**的实现方案：用 `OncePerRequestFilter` 计时 + 内存聚合器做实时统计 + 提供一个导出 CSV 的接口（也可定时落盘）。

# 方案概览

* **计时位置**：`OncePerRequestFilter`（覆盖整个请求，包括控制器、拦截器、视图渲染；并处理异步请求）。
* **标识 API**：使用 `HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE` 拿到路由模板（如 `/users/{id}`），避免把 path 里的动态 ID 打散成很多条。
* **聚合维度**：按 `HTTP 方法 + 路由模板` 聚合；记录 `count/total/avg/min/max`，可选**直方图桶**粗略算 p95。
* **导出**：提供 `/internal/metrics/api-report.csv` 下载 CSV；也可暴露 `/internal/metrics/reset` 清零。
* **线程安全**：`ConcurrentHashMap + LongAdder/AtomicLong/LongAccumulator`。
* **零依赖**：不引入 Actuator / Micrometer / OpenCSV / POI 等。

---

## 1) 聚合器

```java
package com.example.metrics;

import java.util.Arrays;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.*;
import java.util.stream.Collectors;

public class ApiMetricsRegistry {

    // 默认的耗时桶（毫秒），用于近似 p95/p99（可按需调整）
    private static final long[] BUCKETS_MS = {1, 5, 10, 25, 50, 100, 250, 500, 1000, 2000, 5000};
    private static final long[] BUCKETS_NS = Arrays.stream(BUCKETS_MS).map(ms -> ms * 1_000_000L).toArray();

    public static final class ApiKey {
        public final String method;
        public final String pattern;

        public ApiKey(String method, String pattern) {
            this.method = method;
            this.pattern = pattern;
        }
        @Override public int hashCode() { return (method + " " + pattern).hashCode(); }
        @Override public boolean equals(Object o) {
            if (!(o instanceof ApiKey)) return false;
            ApiKey k = (ApiKey) o;
            return method.equals(k.method) && pattern.equals(k.pattern);
        }
        @Override public String toString() { return method + " " + pattern; }
    }

    public static final class ApiTimer {
        private final LongAdder count = new LongAdder();
        private final LongAdder totalNanos = new LongAdder();
        private final LongAccumulator maxNanos = new LongAccumulator(Long::max, 0L);
        private final AtomicLong minNanos = new AtomicLong(Long.MAX_VALUE);

        // 状态码段计数
        private final LongAdder s2xx = new LongAdder();
        private final LongAdder s4xx = new LongAdder();
        private final LongAdder s5xx = new LongAdder();

        // 直方图桶
        private final LongAdder[] buckets = Arrays.stream(BUCKETS_NS).mapToObj(x -> new LongAdder()).toArray(LongAdder[]::new);
        private final LongAdder bucketOverflow = new LongAdder();

        public void record(long durationNanos, int status) {
            count.increment();
            totalNanos.add(durationNanos);
            maxNanos.accumulate(durationNanos);
            minNanos.getAndUpdate(prev -> Math.min(prev, durationNanos));

            int sc = status / 100;
            if (sc == 2) s2xx.increment();
            else if (sc == 4) s4xx.increment();
            else if (sc == 5) s5xx.increment();

            // 记录到桶
            boolean placed = false;
            for (int i = 0; i < BUCKETS_NS.length; i++) {
                if (durationNanos <= BUCKETS_NS[i]) {
                    buckets[i].increment();
                    placed = true;
                    break;
                }
            }
            if (!placed) bucketOverflow.increment();
        }

        public Snapshot snapshot() {
            long c = count.sum();
            long total = totalNanos.sum();
            long max = maxNanos.get();
            long min = minNanos.get() == Long.MAX_VALUE ? 0L : minNanos.get();
            long[] hist = Arrays.stream(buckets).mapToLong(LongAdder::sum).toArray();
            long overflow = bucketOverflow.sum();
            return new Snapshot(c, total, min, max, hist, overflow, s2xx.sum(), s4xx.sum(), s5xx.sum());
        }
    }

    public static final class Snapshot {
        public final long count, totalNanos, minNanos, maxNanos;
        public final long[] hist; public final long overflow;
        public final long count2xx, count4xx, count5xx;
        Snapshot(long count, long totalNanos, long minNanos, long maxNanos,
                 long[] hist, long overflow, long c2, long c4, long c5) {
            this.count = count; this.totalNanos = totalNanos;
            this.minNanos = minNanos; this.maxNanos = maxNanos;
            this.hist = hist; this.overflow = overflow;
            this.count2xx = c2; this.count4xx = c4; this.count5xx = c5;
        }

        public double avgMs() { return count == 0 ? 0d : (totalNanos / 1_000_000.0) / count; }

        // 用直方图近似 pXX，返回“上界（毫秒）”
        public long approxPercentileMs(double p) {
            if (count == 0) return 0;
            long need = (long)Math.ceil(count * p);
            long acc = 0;
            for (int i = 0; i < hist.length; i++) {
                acc += hist[i];
                if (acc >= need) return BUCKETS_MS[i];
            }
            return BUCKETS_MS[BUCKETS_MS.length - 1] + 1; // 溢出桶
        }
    }

    private final Map<ApiKey, ApiTimer> map = new ConcurrentHashMap<>();

    public void record(ApiKey key, long nanos, int status) {
        map.computeIfAbsent(key, k -> new ApiTimer()).record(nanos, status);
    }

    public Map<ApiKey, Snapshot> snapshotAll() {
        return map.entrySet().stream().collect(Collectors.toMap(
            Map.Entry::getKey, e -> e.getValue().snapshot()
        ));
    }

    public void reset() { map.clear(); }
}
```

---

## 2) 计时 Filter（含异步请求支持）

```java
package com.example.metrics;

import jakarta.servlet.*;
import jakarta.servlet.annotation.WebFilter;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;
import org.springframework.web.servlet.HandlerMapping;

import java.io.IOException;

@Component
@WebFilter(urlPatterns = "/*")
public class ApiTimingFilter extends OncePerRequestFilter {

    private static final String START_ATTR = ApiTimingFilter.class.getName() + ".START_NS";
    private final ApiMetricsRegistry registry;

    public ApiTimingFilter(ApiMetricsRegistry registry) {
        this.registry = registry;
    }

    @Override
    protected boolean shouldNotFilterErrorDispatch() { return false; }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {

        request.setAttribute(START_ATTR, System.nanoTime());

        try {
            chain.doFilter(request, response);
        } finally {
            if (request.isAsyncStarted()) {
                // 异步：在完成时记录
                request.getAsyncContext().addListener(new AsyncListener() {
                    @Override public void onComplete(AsyncEvent event) {
                        record(event.getSuppliedRequest(), (HttpServletResponse) event.getSuppliedResponse());
                    }
                    @Override public void onTimeout(AsyncEvent event) {}
                    @Override public void onError(AsyncEvent event) { record(event.getSuppliedRequest(), (HttpServletResponse) event.getSuppliedResponse()); }
                    @Override public void onStartAsync(AsyncEvent event) {}
                });
            } else {
                // 普通同步请求
                record(request, response);
            }
        }
    }

    private void record(ServletRequest req, HttpServletResponse resp) {
        Object startObj = req.getAttribute(START_ATTR);
        if (!(startObj instanceof Long)) return;
        long start = (Long) startObj;
        long duration = System.nanoTime() - start;

        HttpServletRequest httpReq = (HttpServletRequest) req;

        // 获取路由模板（如 /users/{id}）
        String pattern = (String) httpReq.getAttribute(HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE);
        if (pattern == null) pattern = httpReq.getRequestURI();

        String method = httpReq.getMethod();
        int status = resp.getStatus();

        registry.record(new ApiMetricsRegistry.ApiKey(method, pattern), duration, status);
    }
}
```

> 说明：`OncePerRequestFilter` 已经帮你处理了多次 `FORWARD/ERROR` 派发；我们显式开启 `shouldNotFilterErrorDispatch=false` 以便异常路径也统计到。

**配置 Bean：**

```java
package com.example.metrics;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MetricsConfig {
    @Bean
    public ApiMetricsRegistry apiMetricsRegistry() {
        return new ApiMetricsRegistry();
    }
}
```

---

## 3) 导出 CSV & 管理接口

```java
package com.example.metrics;

import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.nio.charset.StandardCharsets;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.Map;

@RestController
@RequestMapping("/internal/metrics")
public class ApiMetricsController {

    private final ApiMetricsRegistry registry;

    public ApiMetricsController(ApiMetricsRegistry registry) {
        this.registry = registry;
    }

    @GetMapping(value = "/api-report.csv", produces = "text/csv")
    public ResponseEntity<byte[]> exportCsv() {
        Map<ApiMetricsRegistry.ApiKey, ApiMetricsRegistry.Snapshot> data = registry.snapshotAll();

        StringBuilder sb = new StringBuilder();
        // CSV 头
        sb.append("method,pattern,count,avg_ms,p50_ms,p95_ms,min_ms,max_ms,total_ms,2xx,4xx,5xx\n");

        data.forEach((k, snap) -> {
            double avgMs = snap.avgMs();
            long p50 = snap.approxPercentileMs(0.50);
            long p95 = snap.approxPercentileMs(0.95);
            long minMs = Math.round(snap.minNanos / 1_000_000.0);
            long maxMs = Math.round(snap.maxNanos / 1_000_000.0);
            long totalMs = Math.round(snap.totalNanos / 1_000_000.0);

            // 简单 CSV 转义（双引号包裹并转义内部双引号）
            String method = csv(k.method);
            String pattern = csv(k.pattern);

            sb.append(method).append(',')
              .append(pattern).append(',')
              .append(snap.count).append(',')
              .append(formatDouble(avgMs)).append(',')
              .append(p50).append(',')
              .append(p95).append(',')
              .append(minMs).append(',')
              .append(maxMs).append(',')
              .append(totalMs).append(',')
              .append(snap.count2xx).append(',')
              .append(snap.count4xx).append(',')
              .append(snap.count5xx).append('\n');
        });

        byte[] bytes = sb.toString().getBytes(StandardCharsets.UTF_8);
        String ts = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMdd-HHmmss"));
        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"api-report-" + ts + ".csv\"")
                .contentType(MediaType.valueOf("text/csv; charset=utf-8"))
                .body(bytes);
    }

    @PostMapping("/reset")
    public String reset() {
        registry.reset();
        return "ok";
    }

    // --- helpers ---
    private static String csv(String v) {
        String s = v == null ? "" : v;
        s = s.replace("\"", "\"\"");
        return "\"" + s + "\"";
    }

    private static String formatDouble(double d) {
        return String.format(java.util.Locale.US, "%.3f", d);
    }
}
```

> 这样你就能直接用浏览器/脚本访问：`GET /internal/metrics/api-report.csv` 下载；Excel/Numbers/Sheets 都能直接打开。

---

## 4)（可选）定时落盘

如果你想每天生成一份文件（不想手动下载），加一个 `@Scheduled` 即可（仍然零依赖）：

```java
package com.example.metrics;

import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.io.*;
import java.nio.charset.StandardCharsets;
import java.time.*;
import java.time.format.DateTimeFormatter;
import java.util.Map;

@EnableScheduling
@Component
public class ApiMetricsDumper {

    private final ApiMetricsRegistry registry;

    public ApiMetricsDumper(ApiMetricsRegistry registry) { this.registry = registry; }

    // 每日 02:00 导出到本地（Asia/Tokyo）
    @Scheduled(cron = "0 0 2 * * *", zone = "Asia/Tokyo")
    public void dump() throws IOException {
        Map<ApiMetricsRegistry.ApiKey, ApiMetricsRegistry.Snapshot> data = registry.snapshotAll();
        String ts = LocalDateTime.now(ZoneId.of("Asia/Tokyo")).format(DateTimeFormatter.ofPattern("yyyyMMdd-HHmmss"));
        File file = new File("api-report-" + ts + ".csv");

        try (Writer w = new OutputStreamWriter(new FileOutputStream(file), StandardCharsets.UTF_8)) {
            w.write("method,pattern,count,avg_ms,p50_ms,p95_ms,min_ms,max_ms,total_ms,2xx,4xx,5xx\n");
            for (var e : data.entrySet()) {
                var k = e.getKey();
                var s = e.getValue();
                long p50 = s.approxPercentileMs(0.50), p95 = s.approxPercentileMs(0.95);
                long minMs = Math.round(s.minNanos / 1_000_000.0);
                long maxMs = Math.round(s.maxNanos / 1_000_000.0);
                long totalMs = Math.round(s.totalNanos / 1_000_000.0);
                w.write(String.format(java.util.Locale.US,
                        "\"%s\",\"%s\",%d,%.3f,%d,%d,%d,%d,%d,%d,%d,%d\n",
                        k.method.replace("\"","\"\""), k.pattern.replace("\"","\"\""),
                        s.count, s.avgMs(), p50, p95, minMs, maxMs, totalMs, s.count2xx, s.count4xx, s.count5xx));
            }
        }
        // 如果希望“滚动”而非累计，可在这里清零
        // registry.reset();
    }
}
```

---

## 注意点 & 小优化

* **异步**：已通过 `AsyncListener` 统计 `Callable/DeferredResult/WebAsyncTask` 等异步请求的真正完成时长。
* **路径模板**：通过 `BEST_MATCHING_PATTERN_ATTRIBUTE` 归一化动态路径，避免大量 label 爆炸。
* **直方图近似**：p50/p95 是桶近似值，足够做运维排查；要精准分位数需更复杂的数据结构（会增加实现成本）。
* **内存占用**：每个“方法+模板”维护一份计数器，数量通常可控；如还担心，可加白名单或忽略静态资源路径。
* **安全**：把导出/重置接口挂到内部路径（如 `/internal/metrics/...`），并加上内网/鉴权限制。
* **StopWatch 的使用**：这里直接 `System.nanoTime()`，更轻量；若你偏好 `StopWatch`，也可以在 filter 中创建本地 `StopWatch`，本质一样。

---

**总结**：这个方案完全在 Spring Boot 内解决了“统计每个 API 请求耗时并导出报告”的需求，生产可用、成本很低、无第三方依赖。你可以先按上面代码接入，跑一两天后下载 CSV 看看分布，再决定是否要加更细的桶或拆分更多维度（如按状态码细分、按业务标签等）。
