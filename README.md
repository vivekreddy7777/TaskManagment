var currentPath = Environment.GetEnvironmentVariable("PATH") ?? "";
var cliPath = Path.Combine(AppContext.BaseDirectory, "clidriver", "bin");

// Add clidriver\bin to PATH if not already present
if (!currentPath.Split(';').Any(p => 
        string.Equals(p.TrimEnd('\\'), cliPath.TrimEnd('\\'), StringComparison.OrdinalIgnoreCase)))
{
    Environment.SetEnvironmentVariable("PATH", currentPath + ";" + cliPath, EnvironmentVariableTarget.Process);
}

// Optional: sanity check log
if (!File.Exists(Path.Combine(cliPath, "db2app64.dll")))
{
    throw new DllNotFoundException($"db2app64.dll not found in {cliPath}");
}
