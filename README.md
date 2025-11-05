
public async Task<CompareToolJobModel> LoadInputFileAsync(CompareToolJobModel job, CancellationToken ct = default)
{
    _logger.LogInformation("LoadInputFile started");

    // --- Resolve work root (fallback to app directory) ---
    string? workRoot = await _systemVariableService.GetAsync("WorkFolder", "ALL");
    workRoot = string.IsNullOrWhiteSpace(workRoot) ? BTAServiceHost.AppDirectory : workRoot;

    // Make sure the actual folder (not only the drive root) exists
    Directory.CreateDirectory(workRoot);

    // Optional guard: if the path has a drive letter that doesn't exist, fail early
    var driveRoot = Path.GetPathRoot(workRoot);
    if (!string.IsNullOrEmpty(driveRoot) &&
        driveRoot.EndsWith(@":\", StringComparison.Ordinal) &&
        !Directory.Exists(driveRoot))
    {
        throw new DirectoryNotFoundException($"Work drive not found: {driveRoot}");
    }

    if (string.IsNullOrWhiteSpace(job.InputSourcePath))
        throw new InvalidOperationException("InputSourcePath is not set.");

    // --- Normalize input path (map F:\Intranet → UNC) ---
    if (job.InputSourcePath.StartsWith(@"F:\Intranet", StringComparison.OrdinalIgnoreCase))
    {
        const string serverName = @"\\CH3MP0110685";
        var relative = job.InputSourcePath.Substring(@"F:\Intranet".Length).TrimStart('\\');
        job.InputSourcePath = Path.Combine(serverName, "Intranet", relative);
    }

    var fileNameNoExt = Path.GetFileNameWithoutExtension(job.InputSourcePath);
    var extFromInput  = Path.GetExtension(job.InputSourcePath).TrimStart('.');

    if (!string.Equals(extFromInput, "xlsx", StringComparison.OrdinalIgnoreCase))
    {
        _logger.LogInformation("Input is not XLSX ({Ext}); skipping Excel conversion.", extFromInput);
        return job;
    }

    try
    {
        // 1) Temp Excel (local copy)
        string compareFolder = Path.Combine(workRoot, "Compare");
        Directory.CreateDirectory(compareFolder);

        string tempExcel = Path.Combine(compareFolder, $"{Guid.NewGuid():N}.xlsx");
        File.Copy(job.InputSourcePath, tempExcel, overwrite: true);
        _logger.LogInformation("Copied input Excel: {src} → {dst}", job.InputSourcePath, tempExcel);

        // 2) Convert to TXT next to the Excel
        string tempTxt = Path.ChangeExtension(tempExcel, ".txt");
        _logger.LogInformation("Converting Excel to text {Output}", tempTxt);

        using (var xl = new ExcelObjects(_logger))
        {
            await xl.InitializeAsync();
            xl.OpenWorkbook(tempExcel, readOnly: true);
            xl.SaveAsText(tempTxt);
        }

        // 3) Cleanup Excel
        if (File.Exists(tempExcel))
            File.Delete(tempExcel);

        // 4) Put TXT into the “network” folder (can be the same work root when running locally)
        string tempNetworkFolder = Path.Combine(workRoot, "CompareOut");
        if (!Directory.Exists(tempNetworkFolder))               // <-- FIXED
            Directory.CreateDirectory(tempNetworkFolder);

        string networkTxt = Path.Combine(tempNetworkFolder, Path.GetFileName(tempTxt));
        File.Copy(tempTxt, networkTxt, overwrite: true);

        // optional: cleanup local txt after copying
        // File.Delete(tempTxt);

        job.FileName = networkTxt;
        _logger.LogInformation("Transfer to Network location (Target): {Target}", job.FileName);
        _logger.LogInformation("LoadInputFile End");
        return job;
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "LoadInputFile failed.");
        throw;
    }
}
