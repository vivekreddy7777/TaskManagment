internal static class Db2Bootstrap
{
    [DllImport("kernel32.dll", SetLastError = true)]
    private static extern bool SetDefaultDllDirectories(int flags);

    [DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
    private static extern IntPtr AddDllDirectory(string path);

    private const int LOAD_LIBRARY_SEARCH_DEFAULT_DIRS = 0x00001000;

    public static void Init()
    {
        var db2Dir = Path.Combine(AppContext.BaseDirectory, "clidriver", "bin");
        if (!Directory.Exists(db2Dir))
            throw new DirectoryNotFoundException(db2Dir);

        SetDefaultDllDirectories(LOAD_LIBRARY_SEARCH_DEFAULT_DIRS);
        AddDllDirectory(db2Dir);

        Environment.SetEnvironmentVariable("DB2PATH", db2Dir);
        Environment.SetEnvironmentVariable("PATH", db2Dir + ";" + Environment.GetEnvironmentVariable("PATH"));
    }
}

// In Program.cs:
Db2Bootstrap.Init(); 
