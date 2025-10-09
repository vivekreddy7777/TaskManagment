namespace BTA_CompareTool_Service;

public sealed class CompareToolOptions
{
    // from ConnectionStrings:BTEBDConnectionString (merged configuration)
    public string ConnectionString { get; init; } = string.Empty;

    // CompareTool:* section (merged configuration)
    public string Module           { get; init; } = "CompareTool";
    public string UserId           { get; init; } = string.Empty;
    public string Password         { get; init; } = string.Empty;

    public int MaxThreads          { get; init; } = 4;
    public int ChannelCapacity     { get; init; } = 200;
    public int CommandTimeoutSec   { get; init; } = 60;
    public int ReceiveTimeoutSec   { get; init; } = 10;

    // optional: 0 => clamp to min(CPU, Threads)
    public int JobWorkers          { get; init; } = 0;
}
