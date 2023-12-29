---
layout: post
title: dotnet core 实现MySQL数据库备份和还原
slug: dotnet-core-mysql-backup
category: csharp
date: 2023-10-26
tags:
  - dotnet core
  - Linux
  - mysql
---

某个小项目需要支持一下 MySQL 全库备份，

本来想着要不做个从库之类的完事了，

但是本来数据库也就几万条数据，

搞这玩意实在是蛋疼。

又想着要不直接启定时任务，

用 MySQL dump 命令行折腾一下算了，

但是....

还是觉得懒。

要不看下有没有人封了库？

SO

找到了...

代码

```csharp

Backup/Export a MySQL Database
string constring = "server=localhost;user=root;pwd=qwerty;database=test;";
string file = "C:\\backup.sql";
using (MySqlConnection conn = new MySqlConnection(constring))
{
using (MySqlCommand cmd = new MySqlCommand())
{
using (MySqlBackup mb = new MySqlBackup(cmd))
{
cmd.Connection = conn;
conn.Open();
mb.ExportToFile(file);
conn.Close();
}
}
}

Import/Restore a MySQL Database

string constring = "server=localhost;user=root;pwd=qwerty;database=test;";
string file = "C:\\backup.sql";
using (MySqlConnection conn = new MySqlConnection(constring))
{
using (MySqlCommand cmd = new MySqlCommand())
{
using (MySqlBackup mb = new MySqlBackup(cmd))
{
cmd.Connection = conn;
conn.Open();
mb.ImportFromFile(file);
conn.Close();
}
}
}

代码写得很清晰，反正抄一下就好。

生产可用代码
using System;
using System.Collections.Generic;
using System.Linq;
using System.Net;
using System.Net.Mail;
using System.Text;
using System.Threading.Tasks;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using MySql.Data.MySqlClient;

namespace DatabasePro.Service
{

    public class BackFileInfo
    {
        public int Id { get; set; }

        public string FileName { get; set; }

        public string FilePath { get; set; }

        public string FileSize { get; set; }

        public string FileSizeMB
        {
            get
            {
                var size = Convert.ToDouble(FileSize);
                return (size / 1024 / 1024).ToString("0.00");
            }
        }

        public string CreateTime { get; set; }

    }

    public class DBBackService : IService
    {
        private AppConfiguration configuration;

        private ILogger<DBBackService> _logger;



        public DBBackService(IOptions<AppConfiguration> options, ILogger<DBBackService> logger)
        {
            configuration = options.Value;
            _logger = logger;
        }


        public string ExportToFile()
        {
            string baseDir = configuration.StoredFilesPath + "/backup";
            _logger.LogInformation("Start backup database.");
            var conString = configuration.QCloudMySQL;
            var today = DateTime.Now;
            string importFileName = $"{baseDir}/{today.ToString("yyyyMMdd_HHmm")}.sql";
            _logger.LogInformation($"Backup file name: {importFileName}");

            using (MySqlConnection conn = new MySqlConnection(conString))
            {
                _logger.LogInformation($"ConnectionTimeout:{conn.ConnectionTimeout}");
                using (MySqlCommand cmd = new MySqlCommand())
                {
                    cmd.CommandTimeout = 3600;
                    using (MySqlBackup mb = new MySqlBackup(cmd))
                    {
                        cmd.Connection = conn;
                        conn.Open();
                        mb.ExportToFile(importFileName);
                        conn.Close();
                    }
                }
            }
            _logger.LogInformation($"Backup database to {importFileName} successfully.");
            return importFileName;
        }

        public string ImportFromFile(string filePath)
        {


            var conString = configuration.QCloudMySQL;
            _logger.LogInformation($"ImportFromFile filePath: {filePath} start.");
            using (MySqlConnection conn = new MySqlConnection(conString))
            {
                using (MySqlCommand cmd = new MySqlCommand())
                {
                    cmd.CommandTimeout = 3600;
                    using (MySqlBackup mb = new MySqlBackup(cmd))
                    {
                        cmd.Connection = conn;
                        conn.Open();
                        mb.ImportFromFile(filePath);
                        conn.Close();
                    }
                }
            }
            _logger.LogInformation($"ImportFromFile {filePath} successfully, user: {userInfo?.UserName}");
            return filePath;
        }

        public void DeleteFile(string filePath)
        {
            if (string.IsNullOrEmpty(filePath))
            {
                return;
            }
            if (!filePath.Contains("backup"))
            {
                _logger.LogInformation($"DeleteFile filePath: {filePath} not in backup folder, ignore.");
                return;
            }

            if (System.IO.File.Exists(filePath))
            {
                System.IO.File.Delete(filePath);
            }
            _logger.LogInformation($"DeleteFile {filePath} successfully, user: {userInfo?.UserName}");
        }
        public QueryResult<BackFileInfo> LoadBackupList()
        {
            var fileList = new List<BackFileInfo>();
            string baseDir = configuration.StoredFilesPath + "/backup";
            _logger.LogInformation("Start load backup list.");
            if (!System.IO.Directory.Exists(baseDir))
            {
                System.IO.Directory.CreateDirectory(baseDir);
            }
            var files = System.IO.Directory.GetFiles(baseDir);
            var index = 1;
            foreach (var file in files)
            {
                var fileInfo = new System.IO.FileInfo(file);
                var backFileInfo = new BackFileInfo();
                backFileInfo.Id = index;
                backFileInfo.FileName = fileInfo.Name;
                backFileInfo.FilePath = file;
                backFileInfo.FileSize = fileInfo.Length.ToString();
                backFileInfo.CreateTime = fileInfo.CreationTime.ToString("yyyy-MM-dd HH:mm:ss");
                fileList.Add(backFileInfo);
                index = index + 1;
            }
            var result = new QueryResult<BackFileInfo>();
            result.list = fileList.OrderByDescending(x => x.CreateTime).ToList();
            result.total = fileList.Count;
            return result;
        }
    }

}
```

注释也没写，反正没几行代码。

```csharp
using Microsoft.AspNetCore.Mvc;
using DatabasePro.Filters;
using DatabasePro.RestAPI.Helpers;
using DatabasePro.Service;
using Swashbuckle.AspNetCore.Annotations;

namespace DatabasePro.Controllers
{
[ApiController]
public class DBBackController : ApiControllerBase
{
private readonly DBBackService \_dbBackService;

        public DBBackController(DBBackService dbBackService)
        {
            _dbBackService = dbBackService;
        }


        /// <summary>
        ///
        /// </summary>
        [HttpGet]
        [Route("/v1/admin/db-back-list")]
        [ServiceFilter(typeof(AdminTokenAttribute))]
        [SwaggerResponse(200, "successfully.")]
        public IActionResult LoadBackupList()
        {
            var fileList = _dbBackService.LoadBackupList();
            return Wrapper(fileList);
        }

        /// <summary>
        /// 导出到备份文件
        /// </summary>
        [HttpGet]
        [Route("/v1/admin/db-back-export")]
        [ServiceFilter(typeof(AdminTokenAttribute))]
        public IActionResult ExportToFile()
        {
            var backSQLFile = _dbBackService.ExportToFile();
            return Ok(new { data = backSQLFile, code = 0 });
        }

        /// <summary>
        /// 导入备份文件
        /// </summary>
        [HttpGet]
        [Route("/v1/admin/db-back-import")]
        [ServiceFilter(typeof(AdminTokenAttribute))]
        public IActionResult ExportToFile([FromQuery] string filePath)
        {
            var backSQLFile = _dbBackService.ImportFromFile(filePath);
            return Ok(new { data = backSQLFile, code = 0 });
        }

        /// <summary>
        /// 删除备份文件
        /// </summary>
        [HttpDelete]
        [Route("/v1/admin/db-back-delete")]
        [ServiceFilter(typeof(AdminTokenAttribute))]
        public IActionResult DeleteFile([FromQuery] string filePath)
        {
            _dbBackService.DeleteFile(filePath);
            return Ok(new { data = new { }, code = 0 });
        }


    }

}


```

接口大概是上面，自己照着调整一下完事。

糊个页面

大概，就这样。

彩蛋时间
可能会遇到 MySQL 连接超时错误。

看情况设置一下”connect-timeout“ 参数。

connect-timeout=3600

导入数据的时候，可能会遇到”mysql Packets larger than max_allowed_packet are not allowed.“

需要调整 ”max_allowed_packet“ 参数。

max-allowed-packet=1073741824

如果你的 MySQL 也是 docker-compose 部署的，可以参考这个~

          args:
            - --lower-case-table-names=1
            - --max-connections=4000
            - --connect-timeout=3600
            - --max-allowed-packet=1073741824
            - --sql-mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION

还有彩蛋

如果你想实现一个定时备份，

在 dotnet core 中可以用 IHostService 实现，

代码大概是

```csharp

using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.Extensions.Hosting;
using System.Threading;

namespace DatabasePro.Service
{

    public class DBBackupTimeHostedService : IHostedService, IDisposable
    {
        private readonly DBBackService _dbBackService;

        private Timer _timer;

        public DBBackupTimeHostedService(DBBackService dBBackService)
        {
            _dbBackService = dBBackService;
        }

        public Task StartAsync(CancellationToken cancellationToken)
        {
            // 启动定时器
            _timer = new Timer(DoWork, null, GetDelayTo3AM(), TimeSpan.FromDays(1));
            Serilog.Log.Information($"DBBackupTimeHostedService: StartAsync");
            return Task.CompletedTask;
        }

        private void DoWork(object state)
        {
            try
            {
                var filePath = _dbBackService.ExportToFile();
                Serilog.Log.Information($"DBBackupTimeHostedService: DoWork finish, filePath:{filePath}");
            }
            catch (Exception ex)
            {
                Serilog.Log.Error($"DBBackupTimeHostedService: DoWork error, {ex.Message}");
            }
        }

        public Task StopAsync(CancellationToken cancellationToken)
        {
            // 停止定时器
            _timer?.Change(Timeout.Infinite, 0);
            return Task.CompletedTask;
        }

        public void Dispose()
        {
            _timer?.Dispose();
        }

        // 北京时间凌晨3点执行（UTC 19点）
        private TimeSpan GetDelayTo3AM()
        {
            DateTime nowInUTC = DateTime.UtcNow;
            DateTime next7PMInUTC = nowInUTC.Date.AddHours(19);

            if (nowInUTC.Hour >= 19)
            {
                next7PMInUTC = next7PMInUTC.AddDays(1);
            }
            return next7PMInUTC - nowInUTC;
        }
    }

}
services.AddHostedService<DBBackupTimeHostedService>();

```

完事。

结尾

简单粗暴的方案总有风险，

苟住苟住~
