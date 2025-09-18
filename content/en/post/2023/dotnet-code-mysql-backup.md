---
title: MySQL Full Backup and Restore in .NET Core
slug: dotnet-core-mysql-backup
date: 2023-10-26 23:09:00+0000
tags:
  - dotnet core
---

A small project needed MySQL full DB backups. Instead of setting up replicas or cron + mysqldump, I looked for a library and found MySqlBackup.

Snippet (from docs):

```csharp
// Export
string constring = "server=localhost;user=root;pwd=qwerty;database=test;";
string file = "C:\\backup.sql";
using (var conn = new MySqlConnection(constring))
using (var cmd = new MySqlCommand())
using (var mb = new MySqlBackup(cmd))
{
    cmd.Connection = conn;
    conn.Open();
    mb.ExportToFile(file);
}

// Import
using (var conn = new MySqlConnection(constring))
using (var cmd = new MySqlCommand())
using (var mb = new MySqlBackup(cmd))
{
    cmd.Connection = conn;
    conn.Open();
    mb.ImportFromFile(file);
}
```

Productionized version wraps it in a service with endpoints to export/import/delete and to list backup files. It also includes a hosted service for scheduled nightly backups.

Tips:

- If you hit connection timeouts, raise `connect-timeout` (e.g., 3600).
- For large imports: “Packets larger than max_allowed_packet…” ⇒ increase `max_allowed_packet` (e.g., 1GB).
- docker-compose example args:

```yaml
  args:
    - --lower-case-table-names=1
    - --max-connections=4000
    - --connect-timeout=3600
    - --max-allowed-packet=1073741824
    - --sql-mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```

Scheduled backup via `IHostedService` (runs daily at UTC 19:00 / Beijing 03:00):

```csharp
public class DBBackupTimeHostedService : IHostedService, IDisposable
{
    private readonly DBBackService _dbBackService;
    private Timer _timer;
    public DBBackupTimeHostedService(DBBackService svc) { _dbBackService = svc; }

    public Task StartAsync(CancellationToken ct)
    {
        _timer = new Timer(DoWork, null, GetDelayTo3AM(), TimeSpan.FromDays(1));
        return Task.CompletedTask;
    }
    void DoWork(object state)
    {
        try { _dbBackService.ExportToFile(); }
        catch (Exception ex) { /* log */ }
    }
    public Task StopAsync(CancellationToken ct) { _timer?.Change(Timeout.Infinite, 0); return Task.CompletedTask; }
    public void Dispose() => _timer?.Dispose();
    TimeSpan GetDelayTo3AM()
    {
        var now = DateTime.UtcNow;
        var next = now.Date.AddHours(19);
        if (now.Hour >= 19) next = next.AddDays(1);
        return next - now;
    }
}
// services.AddHostedService<DBBackupTimeHostedService>();
```

Simple and effective — mind the risks and keep backups safe.

