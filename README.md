protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    _logger.LogInformation("CompareTool Service Started");

    // keep the scope alive for the duration of a single run
    using var scope = _services.CreateScope();
    var sp = scope.ServiceProvider;

    // 1) spin up the service so ServiceControl loads module details (incl. MainRequestTable)
    var svcLogger = sp.GetRequiredService<ILogger<BTA_Service>>();
    var svc = new BTA_Service(
        module: "CompareTool",
        processor: null,
        serviceProvider: sp,
        logger: svcLogger);

    // after BTA_Service ctor, ServiceControl has read module info from DB
    // expose these however your BTA_Service does (property names may differ)
    var moduleName = svc.ModuleName;              // "CompareTool"
    var tableName  = svc.Control.MainRequestTable; // or svc.TableName/Control.TableName

    // 2) choose what you want to run.
    //    A) run a specific request-number from config (simple one-off)
    //    B) or loop and consume messages from your queue (classic service mode)
    // ---- A) one-off run by RequestNumber from config (optional) ----
    var cfg = sp.GetRequiredService<IConfiguration>();
    var requestNumberStr = cfg["RequestNumber"];  // put in appsettings or user-secrets if you want
    if (!string.IsNullOrWhiteSpace(requestNumberStr) &&
        long.TryParse(requestNumberStr, out var requestNumber))
    {
        // base Job (requestId, module, table, IServiceProvider)
        var baseJob = new Job(requestNumber, moduleName, tableName, sp);

        // derived job that takes baseJob in its ctor (your pattern)
        var compareJob = ActivatorUtilities.CreateInstance<BTA_CompareTool_Job>(sp, baseJob);

        await compareJob.RunAsync(stoppingToken);
        _logger.LogInformation("CompareTool job finished for request {req}.", requestNumber);
        return; // done
    }

    // ---- B) queue-driven loop (simplified skeleton) ----
    while (!stoppingToken.IsCancellationRequested)
    {
        try
        {
            // Your queue receive should come from ServiceControl/BTA_Service.
            // Replace 'TryDequeueAsync' with your real method.
            // It should return either an XML message body OR a request number.
            var message = await svc.TryDequeueAsync(stoppingToken); // <-- implement this on your side

            if (message == null)
            {
                await Task.Delay(TimeSpan.FromSeconds(2), stoppingToken); // idle wait
                continue;
            }

            Job baseJob;

            if (message.RequestNumber.HasValue)
            {
                baseJob = new Job(message.RequestNumber.Value, moduleName, tableName, sp);
            }
            else
            {
                // XML version – your 4.5 path: Job(string xmlMessage, string module, string table)
                baseJob = new Job(message.XmlBody, moduleName, tableName);
                // if your Job(xml,module,table) also needs IServiceProvider, add it to that ctor and pass 'sp'
            }

            var compareJob = ActivatorUtilities.CreateInstance<BTA_CompareTool_Job>(sp, baseJob);
            await compareJob.RunAsync(stoppingToken);

            await svc.CompleteAsync(message, stoppingToken); // tell queue the message was processed
        }
        catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
        {
            break;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled error running CompareTool job.");
            // optional: dead-letter/abandon the message
        }
    }

    _logger.LogInformation("CompareTool Service stopping.");
}
