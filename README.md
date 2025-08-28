public void OpenWorkbook(string filename, bool readOnly = true)
{
    if (string.IsNullOrWhiteSpace(filename))
        throw new ArgumentException("Filename cannot be null or empty", nameof(filename));

    if (!File.Exists(filename))
        throw new FileNotFoundException($"Excel file not found at path: {filename}", filename);

    try
    {
        // Log before opening
        CurrentJob.LogInformation($"Opening Excel Workbook: {filename} (ReadOnly={readOnly})");

        Workbook = Excel.Workbooks.Open(
            filename,
            UpdateLinks: 0,
            ReadOnly: readOnly,
            Format: 5,
            Password: "",
            WriteResPassword: "",
            IgnoreReadOnlyRecommended: true,
            Origin: XlPlatform.xlWindows,
            Delimiter: "\t",
            Editable: false,
            Notify: false,
            Converter: 0,
            AddToMru: true,
            Local: 1,
            CorruptLoad: 0
        );
    }
    catch (Exception ex)
    {
        CurrentJob.LogError(ex, $"Failed to open Excel workbook: {filename}");
        throw;
    }
}

public void SaveAs(string filename)
{
    if (Workbook == null)
        throw new InvalidOperationException("No workbook is open to save.");

    if (string.IsNullOrWhiteSpace(filename))
        throw new ArgumentException("Filename cannot be null or empty", nameof(filename));

    try
    {
        CurrentJob.LogInformation($"Saving Excel workbook to: {filename}");
        Workbook.SaveAs(filename);
    }
    catch (Exception ex)
    {
        CurrentJob.LogError(ex, $"Failed to save Excel workbook: {filename}");
        throw;
    }
}
public void SaveAsText(string filename)
{
    if (Workbook == null)
        throw new InvalidOperationException("No workbook is open to save as text.");

    if (string.IsNullOrWhiteSpace(filename))
        throw new ArgumentException("Filename cannot be null or empty", nameof(filename));

    try
    {
        CurrentJob.LogInformation($"Saving Excel workbook as text to: {filename}");
        Workbook.SaveAs(
            Filename: filename,
            FileFormat: XlFileFormat.xlTextWindows,
            AccessMode: XlSaveAsAccessMode.xlNoChange
        );
    }
    catch (Exception ex)
    {
        CurrentJob.LogError(ex, $"Failed to save Excel workbook as text: {filename}");
        throw;
    }
}
public void Dispose()
{
    try
    {
        CurrentJob.LogInformation("Closing Excel application and releasing COM objects.");

        Workbook?.Close(false);
        Workbook = null;

        Excel?.Quit();
        Excel = null;

        GC.Collect();
        GC.WaitForPendingFinalizers();
    }
    catch (Exception ex)
    {
        CurrentJob.LogError(ex, "Error during Excel cleanup in Dispose()");
    }
}
