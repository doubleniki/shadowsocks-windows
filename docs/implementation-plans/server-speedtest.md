# План реализации встроенного speedtest серверов

## 1. Обзор

Встроенный speedtest позволит пользователям автоматически определять самый быстрый сервер, измерять latency и пропускную способность, что значительно улучшит пользовательский опыт.

## 2. Требования

### Функциональные
- [x] Измерение ping/latency для каждого сервера
- [x] Тестирование пропускной способности (download/upload)
- [x] Автоматический выбор fastest сервера
- [x] История измерений
- [x] Периодическое автоматическое тестирование
- [x] Визуализация результатов
- [ ] Экспорт результатов (опционально)

### Нефункциональные
- Измерение latency < 5 секунд
- Speed test < 30 секунд
- Минимальное влияние на производительность
- Возможность отмены теста

## 3. Архитектура

### 3.1 Компоненты

```text
Models/
├── SpeedTest/
│   ├── PingResult.cs
│   ├── SpeedTestResult.cs
│   └── ServerBenchmark.cs
Services/
├── ISpeedTestService.cs
├── SpeedTestService.cs
├── PingService.cs
└── BandwidthTestService.cs
ViewModels/
├── SpeedTestViewModel.cs
└── ServerBenchmarkViewModel.cs
Views/
└── SpeedTestView.xaml
```

### 3.2 Модели данных

#### PingResult.cs

```csharp
namespace Shadowsocks.Models.SpeedTest
{
    /// <summary>
    /// Result of a ping test
    /// </summary>
    public class PingResult
    {
        /// <summary>
        /// Server ID
        /// </summary>
        public Guid ServerId { get; set; }

        /// <summary>
        /// Average latency in milliseconds
        /// </summary>
        public double AverageLatency { get; set; }

        /// <summary>
        /// Minimum latency
        /// </summary>
        public double MinLatency { get; set; }

        /// <summary>
        /// Maximum latency
        /// </summary>
        public double MaxLatency { get; set; }

        /// <summary>
        /// Standard deviation
        /// </summary>
        public double StandardDeviation { get; set; }

        /// <summary>
        /// Packet loss percentage (0-100)
        /// </summary>
        public double PacketLoss { get; set; }

        /// <summary>
        /// Number of successful pings
        /// </summary>
        public int SuccessfulPings { get; set; }

        /// <summary>
        /// Total number of pings sent
        /// </summary>
        public int TotalPings { get; set; }

        /// <summary>
        /// Timestamp of the test
        /// </summary>
        public DateTime Timestamp { get; set; }

        /// <summary>
        /// Is the server reachable
        /// </summary>
        public bool IsReachable => SuccessfulPings > 0;

        /// <summary>
        /// Quality rating (0-100)
        /// </summary>
        public int QualityRating
        {
            get
            {
                if (!IsReachable) return 0;

                var latencyScore = Math.Max(0, 100 - (AverageLatency / 10));
                var lossScore = 100 - PacketLoss;

                return (int)((latencyScore + lossScore) / 2);
            }
        }
    }
}
```

#### SpeedTestResult.cs

```csharp
namespace Shadowsocks.Models.SpeedTest
{
    /// <summary>
    /// Result of a speed test
    /// </summary>
    public class SpeedTestResult
    {
        /// <summary>
        /// Server ID
        /// </summary>
        public Guid ServerId { get; set; }

        /// <summary>
        /// Ping result
        /// </summary>
        public PingResult Ping { get; set; }

        /// <summary>
        /// Download speed in Mbps
        /// </summary>
        public double DownloadSpeed { get; set; }

        /// <summary>
        /// Upload speed in Mbps
        /// </summary>
        public double UploadSpeed { get; set; }

        /// <summary>
        /// Test duration in seconds
        /// </summary>
        public double TestDuration { get; set; }

        /// <summary>
        /// Timestamp of the test
        /// </summary>
        public DateTime Timestamp { get; set; }

        /// <summary>
        /// Test status
        /// </summary>
        public SpeedTestStatus Status { get; set; }

        /// <summary>
        /// Error message if test failed
        /// </summary>
        public string ErrorMessage { get; set; }

        /// <summary>
        /// Overall score (0-100)
        /// </summary>
        public int OverallScore
        {
            get
            {
                if (Status != SpeedTestStatus.Success)
                    return 0;

                var pingScore = Ping?.QualityRating ?? 0;
                var downloadScore = Math.Min(100, DownloadSpeed * 2); // 50 Mbps = 100 score
                var uploadScore = Math.Min(100, UploadSpeed * 5);    // 20 Mbps = 100 score

                return (int)((pingScore * 0.4) + (downloadScore * 0.4) + (uploadScore * 0.2));
            }
        }
    }

    public enum SpeedTestStatus
    {
        NotStarted,
        Running,
        Success,
        Failed,
        Cancelled
    }
}
```

#### ServerBenchmark.cs

```csharp
namespace Shadowsocks.Models.SpeedTest
{
    /// <summary>
    /// Historical benchmark data for a server
    /// </summary>
    public class ServerBenchmark
    {
        public Guid ServerId { get; set; }
        public Server Server { get; set; }

        /// <summary>
        /// Recent test results (last 30 days)
        /// </summary>
        public List<SpeedTestResult> RecentResults { get; set; } = new();

        /// <summary>
        /// Average latency over recent results (returns NaN if no successful results)
        /// </summary>
        public double AverageLatency
        {
            get
            {
                var successful = RecentResults
                    .Where(r => r.Status == SpeedTestStatus.Success && r.Ping != null)
                    .Select(r => r.Ping.AverageLatency)
                    .ToList();

                return successful.Any() ? successful.Average() : double.NaN;
            }
        }

        /// <summary>
        /// Average download speed (returns NaN if no successful results)
        /// </summary>
        public double AverageDownloadSpeed
        {
            get
            {
                var successful = RecentResults
                    .Where(r => r.Status == SpeedTestStatus.Success)
                    .Select(r => r.DownloadSpeed)
                    .ToList();

                return successful.Any() ? successful.Average() : double.NaN;
            }
        }

        /// <summary>
        /// Latest result
        /// </summary>
        public SpeedTestResult LatestResult =>
            RecentResults.OrderByDescending(r => r.Timestamp).FirstOrDefault();

        /// <summary>
        /// Reliability percentage (successful tests / total tests)
        /// </summary>
        public double Reliability =>
            RecentResults.Count > 0
                ? (double)RecentResults.Count(r => r.Status == SpeedTestStatus.Success) / RecentResults.Count * 100
                : 0;
    }
}
```

---

### 3.3 Services

#### IServerService.cs

```csharp
namespace Shadowsocks.Services
{
    /// <summary>
    /// Service for managing shadowsocks servers
    /// </summary>
    public interface IServerService
    {
        /// <summary>
        /// Get all configured servers
        /// </summary>
        Task<List<Server>> GetAllServersAsync(CancellationToken ct = default);

        /// <summary>
        /// Get server by ID
        /// </summary>
        Task<Server> GetServerByIdAsync(Guid serverId, CancellationToken ct = default);

        /// <summary>
        /// Add a new server
        /// </summary>
        Task<Server> AddServerAsync(Server server, CancellationToken ct = default);

        /// <summary>
        /// Update existing server
        /// </summary>
        Task UpdateServerAsync(Server server, CancellationToken ct = default);

        /// <summary>
        /// Delete a server
        /// </summary>
        Task DeleteServerAsync(Guid serverId, CancellationToken ct = default);

        /// <summary>
        /// Observable for server changes
        /// </summary>
        IObservable<List<Server>> ServersChanged { get; }
    }
}
```

#### ISpeedTestService.cs

```csharp
namespace Shadowsocks.Services
{
    public interface ISpeedTestService
    {
        /// <summary>
        /// Ping a server
        /// </summary>
        Task<PingResult> PingServerAsync(Server server, CancellationToken ct = default);

        /// <summary>
        /// Run full speed test on a server
        /// </summary>
        Task<SpeedTestResult> RunSpeedTestAsync(
            Server server,
            IProgress<SpeedTestProgress> progress = null,
            CancellationToken ct = default);

        /// <summary>
        /// Test all servers
        /// </summary>
        Task<List<SpeedTestResult>> TestAllServersAsync(
            IProgress<SpeedTestProgress> progress = null,
            CancellationToken ct = default);

        /// <summary>
        /// Find fastest server
        /// </summary>
        Task<Server> FindFastestServerAsync(CancellationToken ct = default);

        /// <summary>
        /// Get benchmark history for a server
        /// </summary>
        Task<ServerBenchmark> GetBenchmarkAsync(Guid serverId);

        /// <summary>
        /// Save test result
        /// </summary>
        Task SaveResultAsync(SpeedTestResult result);

        /// <summary>
        /// Observable for test results
        /// </summary>
        IObservable<SpeedTestResult> TestResults { get; }
    }

    public class SpeedTestProgress
    {
        public int CurrentServerIndex { get; set; }
        public int TotalServers { get; set; }
        public Server CurrentServer { get; set; }
        public string Status { get; set; }
        public double PercentComplete { get; set; }
    }
}
```

#### IPingService.cs

```csharp
namespace Shadowsocks.Services
{
    /// <summary>
    /// Service for pinging servers and measuring latency
    /// </summary>
    public interface IPingService
    {
        /// <summary>
        /// Ping a server multiple times and return aggregated results
        /// </summary>
        /// <param name="host">Server hostname or IP address</param>
        /// <param name="port">Server port</param>
        /// <param name="count">Number of ping attempts</param>
        /// <param name="ct">Cancellation token</param>
        /// <returns>Aggregated ping results including average latency and packet loss</returns>
        Task<PingResult> PingAsync(
            string host,
            int port,
            int count = 4,
            CancellationToken ct = default);
    }
}
```

#### IBandwidthTestService.cs

```csharp
namespace Shadowsocks.Services
{
    /// <summary>
    /// Service for measuring download and upload bandwidth
    /// </summary>
    public interface IBandwidthTestService
    {
        /// <summary>
        /// Measure download speed through a proxy server
        /// </summary>
        /// <param name="server">Server to test</param>
        /// <param name="progress">Progress reporter for real-time speed updates</param>
        /// <param name="ct">Cancellation token</param>
        /// <returns>Download speed in Mbps</returns>
        Task<double> MeasureDownloadSpeedAsync(
            Server server,
            IProgress<double> progress = null,
            CancellationToken ct = default);

        /// <summary>
        /// Measure upload speed through a proxy server
        /// </summary>
        /// <param name="server">Server to test</param>
        /// <param name="progress">Progress reporter for real-time speed updates</param>
        /// <param name="ct">Cancellation token</param>
        /// <returns>Upload speed in Mbps</returns>
        Task<double> MeasureUploadSpeedAsync(
            Server server,
            IProgress<double> progress = null,
            CancellationToken ct = default);
    }
}
```

#### PingService.cs

```csharp
namespace Shadowsocks.Services
{
    public class PingService : IPingService
    {
        private readonly ILogger<PingService> _logger;
        private const int DefaultPingCount = 4;
        private const int PingTimeout = 3000; // 3 seconds

        public PingService(ILogger<PingService> logger)
        {
            _logger = logger;
        }

        public async Task<PingResult> PingAsync(
            string host,
            int port,
            int count = DefaultPingCount,
            CancellationToken ct = default)
        {
            var latencies = new List<double>();
            var successful = 0;

            for (int i = 0; i < count; i++)
            {
                if (ct.IsCancellationRequested)
                    break;

                try
                {
                    var sw = Stopwatch.StartNew();

                    using var tcpClient = new TcpClient();
                    var connectTask = tcpClient.ConnectAsync(host, port);
                    var timeoutTask = Task.Delay(PingTimeout, ct);

                    var completedTask = await Task.WhenAny(connectTask, timeoutTask);

                    if (completedTask == connectTask && !connectTask.IsFaulted)
                    {
                        sw.Stop();
                        var latency = sw.Elapsed.TotalMilliseconds;
                        latencies.Add(latency);
                        successful++;

                        _logger.LogDebug("Ping to {Host}:{Port} - {Latency}ms", host, port, latency);
                    }
                    else
                    {
                        _logger.LogDebug("Ping to {Host}:{Port} timed out", host, port);
                    }
                }
                catch (Exception ex)
                {
                    _logger.LogDebug(ex, "Ping to {Host}:{Port} failed", host, port);
                }

                // Wait between pings
                if (i < count - 1)
                    await Task.Delay(500, ct);
            }

            var result = new PingResult
            {
                SuccessfulPings = successful,
                TotalPings = count,
                PacketLoss = ((count - successful) / (double)count) * 100,
                Timestamp = DateTime.UtcNow
            };

            if (latencies.Any())
            {
                result.AverageLatency = latencies.Average();
                result.MinLatency = latencies.Min();
                result.MaxLatency = latencies.Max();
                result.StandardDeviation = CalculateStandardDeviation(latencies);
            }

            return result;
        }

        private double CalculateStandardDeviation(List<double> values)
        {
            var avg = values.Average();
            var sumOfSquares = values.Sum(v => Math.Pow(v - avg, 2));
            return Math.Sqrt(sumOfSquares / values.Count);
        }
    }
}
```

#### BandwidthTestService.cs

```csharp
namespace Shadowsocks.Services
{
    public class BandwidthTestService : IBandwidthTestService
    {
        private readonly ILogger<BandwidthTestService> _logger;
        private readonly IConfiguration _configuration;
        private readonly HttpClient _httpClient;

        private const int TestDurationSeconds = 10;

        public BandwidthTestService(
            ILogger<BandwidthTestService> logger,
            IConfiguration configuration,
            IHttpClientFactory httpClientFactory)
        {
            _logger = logger;
            _configuration = configuration;
            _httpClient = httpClientFactory.CreateClient("SpeedTest");
        }

        /// <summary>
        /// Get test URL from configuration with fallback to default public endpoints
        /// </summary>
        private string GetTestUrl(int sizeInMB = 10)
        {
            // Check configuration first
            var configuredUrl = _configuration["SpeedTest:DownloadUrl"];
            if (!string.IsNullOrEmpty(configuredUrl))
                return configuredUrl;

            // Fallback to public speedtest endpoints
            // Option 1: Cloudflare (reliable, global CDN)
            // Generate random file to prevent caching
            var randomParam = Guid.NewGuid().ToString("N");
            return $"https://speed.cloudflare.com/__down?bytes={sizeInMB * 1024 * 1024}&r={randomParam}";

            // Option 2: Fast.com (Netflix CDN) - requires their API
            // return "https://api.fast.com/netflix/speedtest";

            // Option 3: Your own CDN endpoint
            // return _configuration["SpeedTest:CustomEndpoint"];
        }

        /// <summary>
        /// Get upload test URL from configuration
        /// </summary>
        private string GetUploadUrl()
        {
            // Check configuration first
            var configuredUrl = _configuration["SpeedTest:UploadUrl"];
            if (!string.IsNullOrEmpty(configuredUrl))
                return configuredUrl;

            // Fallback to Cloudflare upload endpoint
            var randomParam = Guid.NewGuid().ToString("N");
            return $"https://speed.cloudflare.com/__up?r={randomParam}";

            // Alternative: Use httpbin.org for testing (not recommended for production)
            // return "https://httpbin.org/post";
        }

        public async Task<double> MeasureDownloadSpeedAsync(
            Server server,
            IProgress<double> progress = null,
            CancellationToken ct = default)
        {
            try
            {
                // Use pre-configured HttpClient from factory with proxy settings
                // Note: Proxy configuration should be set up during service registration
                var sw = Stopwatch.StartNew();
                long totalBytes = 0;

                // Download test data from configured endpoint
                var testUrl = GetTestUrl(10); // 10 MB test file
                _logger.LogDebug("Starting download speed test using {Url}", testUrl);

                var response = await _httpClient.GetAsync(testUrl, HttpCompletionOption.ResponseHeadersRead, ct);
                response.EnsureSuccessStatusCode();

                using var stream = await response.Content.ReadAsStreamAsync(ct);
                var buffer = new byte[8192];
                int bytesRead;

                while ((bytesRead = await stream.ReadAsync(buffer, ct)) > 0)
                {
                    totalBytes += bytesRead;

                    // Report progress
                    var elapsedSeconds = sw.Elapsed.TotalSeconds;
                    if (elapsedSeconds > 0)
                    {
                        var speedMbps = (totalBytes * 8) / (elapsedSeconds * 1_000_000);
                        progress?.Report(speedMbps);
                    }

                    // Stop after test duration
                    if (sw.Elapsed.TotalSeconds >= TestDurationSeconds)
                        break;
                }

                sw.Stop();

                // Calculate final speed in Mbps
                var finalSpeed = (totalBytes * 8) / (sw.Elapsed.TotalSeconds * 1_000_000);

                _logger.LogInformation(
                    "Download speed test completed: {Speed:F2} Mbps ({Bytes} bytes in {Duration:F2}s)",
                    finalSpeed, totalBytes, sw.Elapsed.TotalSeconds);

                return finalSpeed;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Download speed test failed");
                return 0;
            }
        }

        public async Task<double> MeasureUploadSpeedAsync(
            Server server,
            IProgress<double> progress = null,
            CancellationToken ct = default)
        {
            try
            {
                // Use pre-configured HttpClient from factory with proxy settings

                // Generate random data to upload
                var uploadData = new byte[1024 * 1024]; // 1 MB
                Random.Shared.NextBytes(uploadData);

                var sw = Stopwatch.StartNew();
                long totalBytes = 0;

                var uploadUrl = GetUploadUrl();
                _logger.LogDebug("Starting upload speed test using {Url}", uploadUrl);

                // Upload test
                for (int i = 0; i < TestDurationSeconds && !ct.IsCancellationRequested; i++)
                {
                    var content = new ByteArrayContent(uploadData);
                    var response = await _httpClient.PostAsync(uploadUrl, content, ct);

                    if (response.IsSuccessStatusCode)
                    {
                        totalBytes += uploadData.Length;

                        var elapsedSeconds = sw.Elapsed.TotalSeconds;
                        if (elapsedSeconds > 0)
                        {
                            var speedMbps = (totalBytes * 8) / (elapsedSeconds * 1_000_000);
                            progress?.Report(speedMbps);
                        }
                    }
                }

                sw.Stop();

                var finalSpeed = (totalBytes * 8) / (sw.Elapsed.TotalSeconds * 1_000_000);

                _logger.LogInformation(
                    "Upload speed test completed: {Speed:F2} Mbps ({Bytes} bytes in {Duration:F2}s)",
                    finalSpeed, totalBytes, sw.Elapsed.TotalSeconds);

                return finalSpeed;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Upload speed test failed");
                return 0;
            }
        }
    }
}
```

#### SpeedTestService.cs

```csharp
namespace Shadowsocks.Services
{
    public class SpeedTestService : ISpeedTestService
    {
        private readonly IPingService _pingService;
        private readonly IBandwidthTestService _bandwidthService;
        private readonly IServerService _serverService;
        private readonly ILogger<SpeedTestService> _logger;
        private readonly Subject<SpeedTestResult> _testResults;
        private readonly SemaphoreSlim _fileLock = new(1, 1);

        public IObservable<SpeedTestResult> TestResults => _testResults;

        public SpeedTestService(
            IPingService pingService,
            IBandwidthTestService bandwidthService,
            IServerService serverService,
            ILogger<SpeedTestService> logger)
        {
            _pingService = pingService;
            _bandwidthService = bandwidthService;
            _serverService = serverService;
            _logger = logger;
            _testResults = new Subject<SpeedTestResult>();
        }

        public async Task<PingResult> PingServerAsync(Server server, CancellationToken ct = default)
        {
            _logger.LogInformation("Pinging server {Server}", server.FriendlyName);

            var result = await _pingService.PingAsync(server.Server, server.ServerPort, ct: ct);
            result.ServerId = server.Id;

            return result;
        }

        public async Task<SpeedTestResult> RunSpeedTestAsync(
            Server server,
            IProgress<SpeedTestProgress> progress = null,
            CancellationToken ct = default)
        {
            var result = new SpeedTestResult
            {
                ServerId = server.Id,
                Status = SpeedTestStatus.Running,
                Timestamp = DateTime.UtcNow
            };

            var sw = Stopwatch.StartNew();

            try
            {
                _logger.LogInformation("Starting speed test for server {Server}", server.FriendlyName);

                // Step 1: Ping test
                progress?.Report(new SpeedTestProgress
                {
                    CurrentServer = server,
                    Status = "Testing latency...",
                    PercentComplete = 0
                });

                result.Ping = await PingServerAsync(server, ct);

                if (!result.Ping.IsReachable)
                {
                    throw new Exception("Server is not reachable");
                }

                progress?.Report(new SpeedTestProgress
                {
                    CurrentServer = server,
                    Status = "Latency test completed",
                    PercentComplete = 33
                });

                // Step 2: Download test
                progress?.Report(new SpeedTestProgress
                {
                    CurrentServer = server,
                    Status = "Testing download speed...",
                    PercentComplete = 33
                });

                var downloadProgress = new Progress<double>(speed =>
                {
                    progress?.Report(new SpeedTestProgress
                    {
                        CurrentServer = server,
                        Status = $"Download: {speed:F2} Mbps",
                        PercentComplete = 33
                    });
                });

                result.DownloadSpeed = await _bandwidthService.MeasureDownloadSpeedAsync(
                    server, downloadProgress, ct);

                progress?.Report(new SpeedTestProgress
                {
                    CurrentServer = server,
                    Status = "Download test completed",
                    PercentComplete = 66
                });

                // Step 3: Upload test
                progress?.Report(new SpeedTestProgress
                {
                    CurrentServer = server,
                    Status = "Testing upload speed...",
                    PercentComplete = 66
                });

                var uploadProgress = new Progress<double>(speed =>
                {
                    progress?.Report(new SpeedTestProgress
                    {
                        CurrentServer = server,
                        Status = $"Upload: {speed:F2} Mbps",
                        PercentComplete = 66
                    });
                });

                result.UploadSpeed = await _bandwidthService.MeasureUploadSpeedAsync(
                    server, uploadProgress, ct);

                sw.Stop();
                result.TestDuration = sw.Elapsed.TotalSeconds;
                result.Status = SpeedTestStatus.Success;

                _logger.LogInformation(
                    "Speed test completed for {Server}: Ping={Ping}ms, Download={Download:F2}Mbps, Upload={Upload:F2}Mbps",
                    server.FriendlyName, result.Ping.AverageLatency, result.DownloadSpeed, result.UploadSpeed);

                progress?.Report(new SpeedTestProgress
                {
                    CurrentServer = server,
                    Status = "Test completed",
                    PercentComplete = 100
                });
            }
            catch (OperationCanceledException)
            {
                result.Status = SpeedTestStatus.Cancelled;
                result.ErrorMessage = "Test cancelled by user";
                _logger.LogInformation("Speed test cancelled for {Server}", server.FriendlyName);
            }
            catch (Exception ex)
            {
                result.Status = SpeedTestStatus.Failed;
                result.ErrorMessage = ex.Message;
                _logger.LogError(ex, "Speed test failed for {Server}", server.FriendlyName);
            }

            // Publish result
            _testResults.OnNext(result);
            await SaveResultAsync(result);

            return result;
        }

        public async Task<List<SpeedTestResult>> TestAllServersAsync(
            IProgress<SpeedTestProgress> progress = null,
            CancellationToken ct = default)
        {
            var servers = await _serverService.GetAllServersAsync(ct);
            var results = new List<SpeedTestResult>();

            for (int i = 0; i < servers.Count; i++)
            {
                if (ct.IsCancellationRequested)
                    break;

                progress?.Report(new SpeedTestProgress
                {
                    CurrentServerIndex = i,
                    TotalServers = servers.Count,
                    CurrentServer = servers[i],
                    Status = $"Testing server {i + 1} of {servers.Count}",
                    PercentComplete = (i / (double)servers.Count) * 100
                });

                var result = await RunSpeedTestAsync(servers[i], progress, ct);
                results.Add(result);
            }

            return results;
        }

        public async Task<Server> FindFastestServerAsync(CancellationToken ct = default)
        {
            var servers = await _serverService.GetAllServersAsync(ct);

            // Quick ping all servers
            var pingTasks = servers.Select(s => PingServerAsync(s, ct));
            var pingResults = await Task.WhenAll(pingTasks);

            // Find server with best latency
            var fastest = pingResults
                .Where(r => r.IsReachable)
                .OrderBy(r => r.AverageLatency)
                .FirstOrDefault();

            if (fastest != null)
            {
                return servers.First(s => s.Id == fastest.ServerId);
            }

            return null;
        }

        public async Task SaveResultAsync(SpeedTestResult result)
        {
            // Save to database or file with concurrency protection
            var filePath = Path.Combine(
                AppDomain.CurrentDomain.BaseDirectory,
                "speedtest-results.json");

            await _fileLock.WaitAsync();
            try
            {
                var existingResults = new List<SpeedTestResult>();

                if (File.Exists(filePath))
                {
                    var json = await File.ReadAllTextAsync(filePath);
                    existingResults = JsonConvert.DeserializeObject<List<SpeedTestResult>>(json) ?? new();
                }

                existingResults.Add(result);

                // Keep only last 100 results
                if (existingResults.Count > 100)
                {
                    existingResults = existingResults.OrderByDescending(r => r.Timestamp).Take(100).ToList();
                }

                var newJson = JsonConvert.SerializeObject(existingResults, Formatting.Indented);
                await File.WriteAllTextAsync(filePath, newJson);
            }
            finally
            {
                _fileLock.Release();
            }
        }

        public async Task<ServerBenchmark> GetBenchmarkAsync(Guid serverId)
        {
            var filePath = Path.Combine(
                AppDomain.CurrentDomain.BaseDirectory,
                "speedtest-results.json");

            if (!File.Exists(filePath))
                return new ServerBenchmark { ServerId = serverId };

            var json = await File.ReadAllTextAsync(filePath);
            var allResults = JsonConvert.DeserializeObject<List<SpeedTestResult>>(json) ?? new();

            var serverResults = allResults
                .Where(r => r.ServerId == serverId)
                .OrderByDescending(r => r.Timestamp)
                .Take(30)
                .ToList();

            return new ServerBenchmark
            {
                ServerId = serverId,
                RecentResults = serverResults
            };
        }
    }
}
```

---

### 3.4 SpeedTest Endpoint Configuration

#### Overview

The bandwidth testing service requires reliable endpoints for measuring download and upload speeds. This section provides concrete options for configuring speedtest endpoints.

#### Option A: Public Speedtest Endpoints (Recommended for Quick Start)

##### 1. Cloudflare Speed Test (Recommended - Production Ready)

Cloudflare provides free, globally distributed speedtest endpoints:

```json
// appsettings.json
{
  "SpeedTest": {
    "Provider": "Cloudflare",
    "DownloadUrl": "https://speed.cloudflare.com/__down?bytes={size}",
    "UploadUrl": "https://speed.cloudflare.com/__up",
    "TestDurationSeconds": 10,
    "DownloadSizeMB": 10
  }
}
```

**Features:**
- ✅ Free, no API key required
- ✅ Global CDN (200+ locations)
- ✅ High reliability and uptime
- ✅ Supports both download and upload tests
- ✅ No rate limiting for reasonable use

**Usage:**
```csharp
// Download: GET https://speed.cloudflare.com/__down?bytes=10485760
// Upload: POST https://speed.cloudflare.com/__up
```

##### 2. Fast.com API (Netflix CDN)

Fast.com provides a JSON API for speed testing:

```json
{
  "SpeedTest": {
    "Provider": "Fast.com",
    "ApiUrl": "https://api.fast.com/netflix/speedtest/v2",
    "ApiKey": "required" // Obtain from Fast.com
  }
}
```

**Features:**
- ✅ Backed by Netflix CDN
- ✅ Accurate for streaming workloads
- ⚠️ Requires API integration
- ⚠️ Rate limiting may apply

##### 3. LibreSpeed (Open Source)

```json
{
  "SpeedTest": {
    "Provider": "LibreSpeed",
    "BaseUrl": "https://librespeed.org",
    "Endpoints": [
      "https://speedtest1.example.com",
      "https://speedtest2.example.com"
    ]
  }
}
```

Public LibreSpeed instances: [https://github.com/librespeed/speedtest/wiki/Public-Servers](https://github.com/librespeed/speedtest/wiki/Public-Servers)

---

#### Option B: Self-Hosted Speedtest Endpoint

For organizations requiring full control, deploy a dedicated speedtest server.

##### Deployment Guide

##### 1. Using LibreSpeed (Docker)

Create `docker-compose.yml`:
```yaml
version: '3'
services:
  speedtest:
    image: linuxserver/librespeed:latest
    container_name: shadowsocks-speedtest
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=UTC
    ports:
      - "8080:80"
    volumes:
      - ./config:/config
    restart: unless-stopped
```

Deploy:
```bash
docker-compose up -d
```

Configure in Shadowsocks:
```json
{
  "SpeedTest": {
    "DownloadUrl": "http://your-server.com:8080/backend/garbage.php?size={size}",
    "UploadUrl": "http://your-server.com:8080/backend/empty.php"
  }
}
```

##### 2. Using Nginx + Static Files

For simple download testing, serve static files via Nginx:

```nginx
# /etc/nginx/sites-available/speedtest
server {
    listen 80;
    server_name speedtest.your-domain.com;

    location /test-files/ {
        alias /var/www/speedtest/;

        # Disable caching
        add_header Cache-Control "no-store, no-cache, must-revalidate";

        # CORS headers
        add_header Access-Control-Allow-Origin "*";

        # Security headers
        add_header X-Content-Type-Options "nosniff";
    }

    # Upload endpoint (returns 200 OK and discards data)
    location /upload {
        client_max_body_size 100M;

        # Discard upload data
        return 200 "OK";

        add_header Access-Control-Allow-Origin "*";
    }
}
```

Generate test files:
```bash
mkdir -p /var/www/speedtest
dd if=/dev/urandom of=/var/www/speedtest/1mb.bin bs=1M count=1
dd if=/dev/urandom of=/var/www/speedtest/10mb.bin bs=1M count=10
dd if=/dev/urandom of=/var/www/speedtest/100mb.bin bs=1M count=100
```

Configuration:
```json
{
  "SpeedTest": {
    "DownloadUrl": "https://speedtest.your-domain.com/test-files/10mb.bin",
    "UploadUrl": "https://speedtest.your-domain.com/upload"
  }
}
```

##### 3. CDN-Hosted Test Files

Upload test files to your CDN (Cloudflare, AWS CloudFront, Azure CDN):

```bash
# Generate test files locally
for size in 1 10 50 100; do
  dd if=/dev/urandom of=${size}mb.bin bs=1M count=${size}
done

# Upload to S3 (example)
aws s3 cp *.bin s3://your-bucket/speedtest-files/ --acl public-read
```

CloudFront configuration:
```json
{
  "SpeedTest": {
    "DownloadUrl": "https://d1234567890.cloudfront.net/speedtest-files/10mb.bin",
    "UploadUrl": "https://your-api.com/speedtest/upload",
    "CacheBusting": true  // Adds random parameter to prevent caching
  }
}
```

##### Monitoring

Add monitoring to track endpoint health:

```bash
# Prometheus metrics endpoint
curl https://speedtest.your-domain.com/metrics
```

##### Security Considerations

- Enable HTTPS with valid SSL certificate
- Implement rate limiting (e.g., 10 tests per IP per hour)
- Set up monitoring and alerts
- Consider DDoS protection (Cloudflare, AWS Shield)

---

#### Option C: Alternative Measurement Methods

##### 1. Real Traffic Analysis (No External Endpoint Required)

Measure bandwidth by analyzing actual proxy traffic:

```csharp
public class TrafficAnalyzer : ITrafficAnalyzer
{
    private readonly ConcurrentQueue<TrafficSample> _samples = new();

    public void RecordTraffic(long bytes, TimeSpan duration)
    {
        _samples.Enqueue(new TrafficSample
        {
            Bytes = bytes,
            Timestamp = DateTime.UtcNow,
            Duration = duration
        });

        // Keep only last 5 minutes
        while (_samples.TryPeek(out var oldest) &&
               DateTime.UtcNow - oldest.Timestamp > TimeSpan.FromMinutes(5))
        {
            _samples.TryDequeue(out _);
        }
    }

    public double GetAverageThroughput()
    {
        if (_samples.IsEmpty) return 0;

        var totalBytes = _samples.Sum(s => s.Bytes);
        var totalSeconds = _samples.Sum(s => s.Duration.TotalSeconds);

        return (totalBytes * 8) / (totalSeconds * 1_000_000); // Mbps
    }
}
```

##### 2. Integrated Public APIs

Use existing speedtest APIs:

```csharp
// Ookla Speedtest API (requires agreement)
// https://www.speedtest.net/apps/cli

// M-Lab NDT7 Protocol (open source)
// https://www.measurementlab.net/tests/ndt/

public class MLabNdt7Client
{
    public async Task<SpeedTestResult> RunTestAsync()
    {
        // Implement NDT7 WebSocket protocol
        // See: https://github.com/m-lab/ndt-server/blob/master/spec/ndt7-protocol.md
    }
}
```

---

#### Configuration File Examples

**Complete appsettings.json:**

```json
{
  "SpeedTest": {
    // Provider: "Cloudflare", "Fast.com", "Custom", "Traffic"
    "Provider": "Cloudflare",

    // Cloudflare endpoints
    "DownloadUrl": "https://speed.cloudflare.com/__down?bytes={size}",
    "UploadUrl": "https://speed.cloudflare.com/__up",

    // Test parameters
    "TestDurationSeconds": 10,
    "DownloadSizeMB": 10,
    "UploadSizeMB": 5,
    "ParallelConnections": 4,

    // Retry configuration
    "MaxRetries": 3,
    "RetryDelaySeconds": 2,

    // Cache busting
    "CacheBusting": true,
    "CacheBustingParameter": "r",

    // Timeout settings
    "ConnectionTimeoutSeconds": 30,
    "TestTimeoutSeconds": 60,

    // Fallback endpoints (tried in order if primary fails)
    "FallbackEndpoints": [
      {
        "DownloadUrl": "https://speedtest-backup1.example.com/10mb.bin",
        "UploadUrl": "https://speedtest-backup1.example.com/upload"
      },
      {
        "DownloadUrl": "https://speedtest-backup2.example.com/10mb.bin",
        "UploadUrl": "https://speedtest-backup2.example.com/upload"
      }
    ],

    // Traffic-based measurement (alternative to endpoint testing)
    "UseTrafficAnalysis": false,
    "TrafficAnalysisWindowMinutes": 5,

    // Automatic testing
    "AutomaticTesting": {
      "Enabled": false,
      "IntervalHours": 24,
      "TestOnStartup": false
    }
  },

  "Logging": {
    "LogLevel": {
      "Shadowsocks.Services.BandwidthTestService": "Debug"
    }
  }
}
```

**Environment Variables (Docker/Production):**

```bash
# .env file
SPEEDTEST_DOWNLOAD_URL=https://speed.cloudflare.com/__down?bytes={size}
SPEEDTEST_UPLOAD_URL=https://speed.cloudflare.com/__up
SPEEDTEST_DURATION=10
SPEEDTEST_PROVIDER=Cloudflare
```

---

#### Testing and Validation

**Verify endpoint configuration:**

```bash
# Test download endpoint
curl -o /dev/null -w "Time: %{time_total}s\nSpeed: %{speed_download} bytes/s\n" \
  "https://speed.cloudflare.com/__down?bytes=10485760"

# Test upload endpoint
dd if=/dev/zero bs=1M count=10 | curl -X POST \
  -H "Content-Type: application/octet-stream" \
  --data-binary @- \
  "https://speed.cloudflare.com/__up"
```

**Monitor endpoint health:**

```csharp
public class EndpointHealthCheck : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken ct = default)
    {
        try
        {
            using var client = new HttpClient();
            var response = await client.GetAsync(
                "https://speed.cloudflare.com/__down?bytes=1024", ct);

            if (response.IsSuccessStatusCode)
                return HealthCheckResult.Healthy("Speedtest endpoint is reachable");

            return HealthCheckResult.Degraded(
                $"Endpoint returned {response.StatusCode}");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                "Speedtest endpoint is unreachable", ex);
        }
    }
}
```

---

## 4. UI Implementation

### SpeedTestView.xaml

```xml
<UserControl x:Class="Shadowsocks.Views.SpeedTestView"
             xmlns:materialDesign="http://materialdesigninxaml.net/winfx/xaml/themes">

    <Grid Margin="16">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>

        <!-- Toolbar -->
        <StackPanel Grid.Row="0" Orientation="Horizontal" Margin="0,0,0,16">
            <Button Content="Test All Servers"
                    Command="{Binding TestAllServersCommand}"
                    Style="{StaticResource MaterialDesignRaisedButton}"/>

            <Button Content="Find Fastest"
                    Command="{Binding FindFastestCommand}"
                    Margin="8,0,0,0"
                    Style="{StaticResource MaterialDesignRaisedAccentButton}"/>

            <Button Content="Cancel"
                    Command="{Binding CancelTestCommand}"
                    Margin="8,0,0,0"
                    Visibility="{Binding IsTesting, Converter={StaticResource BoolToVisConverter}}"/>

            <!-- Progress -->
            <StackPanel Orientation="Horizontal" Margin="16,0,0,0"
                       Visibility="{Binding IsTesting, Converter={StaticResource BoolToVisConverter}}">
                <ProgressBar Width="200" Height="4"
                            Value="{Binding TestProgress}"
                            Maximum="100"/>
                <TextBlock Text="{Binding TestStatus}" Margin="8,0,0,0" VerticalAlignment="Center"/>
            </StackPanel>
        </StackPanel>

        <!-- Server Results -->
        <ScrollViewer Grid.Row="1">
            <ItemsControl ItemsSource="{Binding ServerBenchmarks}">
                <ItemsControl.ItemTemplate>
                    <DataTemplate>
                        <materialDesign:Card Margin="0,0,0,8" Padding="16">
                            <Grid>
                                <Grid.ColumnDefinitions>
                                    <ColumnDefinition Width="*"/>
                                    <ColumnDefinition Width="Auto"/>
                                    <ColumnDefinition Width="Auto"/>
                                    <ColumnDefinition Width="Auto"/>
                                    <ColumnDefinition Width="Auto"/>
                                </Grid.ColumnDefinitions>

                                <!-- Server Info -->
                                <StackPanel Grid.Column="0">
                                    <TextBlock Text="{Binding Server.FriendlyName}"
                                              Style="{StaticResource MaterialDesignHeadline6TextBlock}"/>
                                    <TextBlock Text="{Binding Server.Server}"
                                              Foreground="{DynamicResource MaterialDesignBodyLight}"/>
                                </StackPanel>

                                <!-- Latency -->
                                <StackPanel Grid.Column="1" Margin="16,0">
                                    <TextBlock Text="Latency" Foreground="{DynamicResource MaterialDesignBodyLight}"/>
                                    <StackPanel Orientation="Horizontal">
                                        <materialDesign:PackIcon Kind="Timer" VerticalAlignment="Center"/>
                                        <TextBlock Text="{Binding AverageLatency, StringFormat={}{0:F0}ms}"
                                                  Margin="4,0,0,0" FontSize="18" FontWeight="Bold"/>
                                    </StackPanel>
                                </StackPanel>

                                <!-- Download Speed -->
                                <StackPanel Grid.Column="2" Margin="16,0">
                                    <TextBlock Text="Download" Foreground="{DynamicResource MaterialDesignBodyLight}"/>
                                    <StackPanel Orientation="Horizontal">
                                        <materialDesign:PackIcon Kind="Download" VerticalAlignment="Center"/>
                                        <TextBlock Text="{Binding AverageDownloadSpeed, StringFormat={}{0:F1} Mbps}"
                                                  Margin="4,0,0,0" FontSize="18" FontWeight="Bold"/>
                                    </StackPanel>
                                </StackPanel>

                                <!-- Overall Score -->
                                <StackPanel Grid.Column="3" Margin="16,0">
                                    <TextBlock Text="Score" Foreground="{DynamicResource MaterialDesignBodyLight}"/>
                                    <Border Background="{Binding ScoreColor}" CornerRadius="16"
                                           Padding="12,4" HorizontalAlignment="Center">
                                        <TextBlock Text="{Binding LatestResult.OverallScore}"
                                                  Foreground="White" FontWeight="Bold" FontSize="18"/>
                                    </Border>
                                </StackPanel>

                                <!-- Actions -->
                                <Button Grid.Column="4"
                                       Content="Test"
                                       Command="{Binding DataContext.TestServerCommand, RelativeSource={RelativeSource AncestorType=UserControl}}"
                                       CommandParameter="{Binding Server}"/>
                            </Grid>
                        </materialDesign:Card>
                    </DataTemplate>
                </ItemsControl.ItemTemplate>
            </ItemsControl>
        </ScrollViewer>
    </Grid>
</UserControl>
```

---

## 5. План реализации

### Неделя 1-2: Модели и базовые сервисы
- [ ] Создать модели (PingResult, SpeedTestResult, ServerBenchmark)
- [ ] Реализовать PingService
- [ ] Написать unit тесты

### Неделя 3-4: Bandwidth testing
- [ ] Реализовать BandwidthTestService
- [ ] Настроить test endpoints
- [ ] Тестирование точности измерений

### Неделя 5-6: SpeedTestService
- [ ] Реализовать SpeedTestService
- [ ] Добавить сохранение результатов
- [ ] Реализовать автоматическое тестирование

### Неделя 7-8: UI
- [ ] Создать SpeedTestViewModel
- [ ] Создать SpeedTestView
- [ ] Добавить визуализацию результатов
- [ ] Интегрировать в главное окно

### Неделя 9-10: Доработка
- [ ] Оптимизация производительности
- [ ] Обработка ошибок
- [ ] История и графики
- [ ] Документация

---

## 6. Метрики успеха

- [ ] Ping test < 5 секунд
- [ ] Full speed test < 30 секунд
- [ ] Точность измерений ±10%
- [ ] Может тестировать 10+ серверов параллельно
- [ ] Визуализация результатов работает корректно

---

**Версия**: 1.0
**Дата**: 2025-11-14
**Статус**: Ready for Implementation
**Приоритет**: Medium
