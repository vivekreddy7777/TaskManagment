/// <summary>
/// Copy the input Excel to a temp location, convert to text, and stage a network file
/// </summary>
public async Task LoadInputFile()
{
    _logger.LogInformation("LoadInputFile started");

    // ---- 0) Sanity ----------------------------------------------------------
    if (string.IsNullOrWhiteSpace(InputSourcePath))
        throw new InvalidOperationException("InputSourcePath is not set.");

    if (!File.Exists(InputSourcePath))
        throw new FileNotFoundException("Input Excel not found", InputSourcePath);

    // ext from input unless FileType is explicitly set
    var fileNameNoExt = Path.GetFileNameWithoutExtension(InputSourcePath);
    var extFromInput  = Path.GetExtension(InputSourcePath)?.TrimStart('.');
    var extToUse      = string.IsNullOrEmpty(FileType) ? extFromInput : FileType;

    // This is where the source would live if we're using a folder
    var pathInSourceLocation = Path.Combine(_sourceFolderLocation, Path.GetFileName(InputSourcePath));

    // ---- 1) Create FileUtilities for the working file -----------------------
    // If the path points to a physical file, use that; otherwise build from folder + name + ext
    var location = File.Exists(InputSourcePath)
        ? new FileUtilities(InputSourcePath)
        : new FileUtilities(_sourceFolderLocation, fileNameNoExt, extToUse);

    // We only convert Excel -> txt if it's xlsx
    if (!string.Equals(extToUse, "xlsx", StringComparison.OrdinalIgnoreCase))
    {
        _logger.LogInformation("Input is not XLSX (.{Ext}); skipping Excel conversion.", extToUse);
        return;
    }

    try
    {
        // ---- 2) Build temp Excel path (local) --------------------------------
        // Example: <work>\Compare\<guid>.xlsx
        string tempExcel = location.BuildFilePath(
            GetWorkFolder("Compare"),
            Constants.NewUniqueKey,
            null,
            "xlsx",
            createFolder: true,
            deleteExistingFile: true
        );

        // ---- 3) Copy real Excel into temp path --------------------------------
        File.Copy(InputSourcePath, tempExcel, overwrite: true);
        _logger.LogInformation("Copied input Excel: {src} -> {dst}", InputSourcePath, tempExcel);

        // ---- 4) Prepare output text file path (local) -------------------------
        // Example: <work>\Compare\<guid>.txt
        location.CurrentFilePath = location.BuildFilePath(
            folder:  null,          // same folder as tempExcel
            filename: null,         // reuse the guid that FileUtilities tracks for NewFilePath/CurrentFilePath
            ext:     "txt",
            createFolder: true,
            deleteExistingFile: false
        );

        // ---- 5) Convert Excel -> Tab-delimited text ---------------------------
        _logger.LogInformation("Converting Excel to text @ {Output}", location.CurrentFilePath);

        using (var xl = new ExcelObjects(this))
        {
            xl.OpenWorkbook(tempExcel, readOnly: true);
            MultiThreadingQueue.UseOleMessageFilter(() =>
            {
                // If your ExcelObjects writes to NewFilePath instead, swap to location.NewFilePath
                xl.SaveAsText(location.CurrentFilePath);
            });
        }

        // ---- 6) Remove local Excel temp --------------------------------------
        FileUtilities.InitializePath(tempExcel, deleteExistingFile: true, createFolder: false);
        _logger.LogInformation("Temp Excel removed: {Temp}", tempExcel);

        // ---- 7) Create network file path (final TXT destination) --------------
        // Example: \\share\...\<guid>.txt   (uses your _tempNetworkFolderLocation)
        string networkTxt = location.BuildFilePath(
            _tempNetworkFolderLocation,
            filename: null,
            ext: "txt",
            createFolder: true,
            deleteExistingFile: true
        );

        // ---- 8) Transfer local TXT -> network --------------------------------
        _logger.LogInformation("Transfer to Network location: {Target}", networkTxt);
        await location.TransferFileAsync(keepLocal: true);   // uses CurrentFilePath => Network(NewFilePath)

        _logger.LogInformation("LoadInputFile End");
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "LoadInputFile failed.");
        throw;
    }
}
