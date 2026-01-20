public class RequestStatusService
{
    private readonly QCDBServerContext _db;
    private readonly ILogger<RequestStatusService> _logger;

    public RequestStatusService(
        QCDBServerContext db,
        ILogger<RequestStatusService> logger)
    {
        _db = db;
        _logger = logger;
    }

    public async Task UpdateRequestStatusAsync(
        int requestNumber,
        string tableName,
        string status,
        string description,
        CancellationToken cancellationToken)
    {
        _logger.LogInformation(
            "Updating request status. RequestNumber={RequestNumber}, Status={Status}",
            requestNumber, status);

        await _db.Database.ExecuteSqlRawAsync(
            @"EXEC dbo.procBTE_System_MainRequest_Status_UPDATE
                @RequestNumber = {0},
                @TableName     = {1},
                @Status        = {2},
                @Description   = {3}",
            new object[]
            {
                requestNumber,
                tableName,
                status,
                description
            },
            cancellationToken);
    }
}
