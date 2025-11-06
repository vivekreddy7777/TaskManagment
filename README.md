// 2. Convert Excel → TXT using ClosedXML (no Excel/Interop required)
var tempTxt = Path.ChangeExtension(tempExcel, ".txt");
_logger.LogInformation("Converting Excel to TXT: {Output}", tempTxt);

try
{
    XlsxToTxt.Convert(tempExcel, tempTxt, _logger);
    _logger.LogInformation("TXT created successfully: {Txt}", tempTxt);
}
catch (Exception ex)
{
    _logger.LogError(ex, "Excel conversion failed. ExcelPath: {ExcelPath}, TxtPath: {TxtPath}", tempExcel, tempTxt);
    throw;
}

// Update job file reference
location.CurrentFilePath = tempTxt;
job.FileName = tempTxt; // for console mode


using ClosedXML.Excel;
using System.Text;

public static class XlsxToTxt
{
    public static void Convert(string xlsxPath, string txtPath, ILogger logger)
    {
        Directory.CreateDirectory(Path.GetDirectoryName(txtPath)!);

        using var wb = new XLWorkbook(xlsxPath);
        var ws = wb.Worksheets.Worksheet(1);

        var range = ws.RangeUsed();
        if (range == null)
            throw new InvalidOperationException("Excel sheet contains no data.");

        var sb = new StringBuilder();

        foreach (var row in range.Rows())
        {
            bool first = true;
            foreach (var cell in row.Cells())
            {
                if (!first) sb.Append('\t');
                first = false;

                var value = cell.GetFormattedString()
                                .Replace("\r", " ")
                                .Replace("\n", " ")
                                .Replace("\t", " ");

                sb.Append(value);
            }
            sb.AppendLine();
        }

        File.WriteAllText(txtPath, sb.ToString(), Encoding.UTF8);

        if (!File.Exists(txtPath))
            throw new IOException($"TXT creation failed: {txtPath}");

        logger.LogInformation("ClosedXML conversion complete.");
    }
}
