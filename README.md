public async Task LoadInputFile()
{
    _logger.LogInformation("LoadInputFile started");

    if (string.IsNullOrWhiteSpace(InputSourcePath))
        throw new InvalidOperationException("InputSourcePath is not set.");

    if (!File.Exists(InputSourcePath))
        throw new FileNotFoundException("Input Excel not found", InputSourcePath);

    var fileNameNoExt = Path.GetFileNameWithoutExtension(InputSourcePath);
    var extFromInput  = Path.GetExtension(InputSourcePath)?.TrimStart('.');
    var extToUse      = string.IsNullOrEmpty(FileType) ? extFromInput : FileType;

    var location = File.Exists(InputSourcePath)
        ? new FileUtilities(InputSourcePath)
        : new FileUtilities(_sourceFolderLocation, fileNameNoExt, extToUse);

    if (!string.Equals(extToUse, "xlsx", StringComparison.OrdinalIgnoreCase))
    {
        _logger.LogInformation("Input is not XLSX (.{Ext}); skipping Excel conversion.", extToUse);
        return;
    }

    try
    {
        // 1. Temp Excel file (local copy)
        string tempExcel = location.BuildFilePath(
            GetWorkFolder("Compare"),
            Constants.NewUniqueKey,
            null,
            "xlsx",
            true,
            false
        );

        File.Copy(InputSourcePath, tempExcel, overwrite: true);
        _logger.LogInformation("Copied input Excel: {src} -> {dst}", InputSourcePath, tempExcel);

        // 2. Temp TXT file
        location.CurrentFilePath = location.BuildFilePath(
            null,
            null,
            "txt",
            true,
            false
        );

        _logger.LogInformation("Converting Excel to text @ {Output}", location.CurrentFilePath);

        using (var xl = new ExcelObjects(this))
        {
            xl.OpenWorkbook(tempExcel, readOnly: true);
            MultiThreadingQueue.UseOleMessageFilter(() =>
            {
                xl.SaveAsText(location.CurrentFilePath);
            });
        }

        // 3. Cleanup Excel
        FileUtilities.InitializePath(tempExcel, true, false);

        // 4. Network TXT file
        string networkTxt = location.BuildFilePath(
            _tempNetworkFolderLocation,
            null,
            "txt",
            true,
            true
        );

        _logger.LogInformation("Transfer to Network location: {Target}", networkTxt);
        await location.TransferFileAsync(true);

        _logger.LogInformation("LoadInputFile End");
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "LoadInputFile failed.");
        throw;
    }
}
