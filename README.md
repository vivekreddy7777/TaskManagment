using System;
using System.Runtime.InteropServices;
using System.Threading.Tasks;
using Microsoft.Extensions.Logging;
using Excel = Microsoft.Office.Interop.Excel; // <-- IMPORTANT

namespace BTA
{
    /// <summary>
    /// Thin wrapper around Excel COM for opening/saving/exporting text.
    /// Make sure the project references Microsoft.Office.Interop.Excel.
    /// </summary>
    public sealed class ExcelObjects : IDisposable
    {
        private readonly ILogger? _logger;

        private Excel.Application? _excel;
        private Excel.Workbook? _workBook;

        public Excel.Application? Application => _excel;
        public Excel.Workbook? Workbook => _workBook;

        public ExcelObjects(ILogger? logger = null)
        {
            _logger = logger;
        }

        /// <summary>Start Excel (no UI, macros disabled).</summary>
        public async Task InitializeAsync()
        {
            TryLog("Starting Excel...");
            await Task.Yield();

            // Start Excel
            _excel = new Excel.Application
            {
                Visible = false,
                DisplayAlerts = false
            };

            // Disable macro automation security (no prompts)
            try
            {
                Excel.AutomationSecurity = Microsoft.Office.Core.MsoAutomationSecurity.msoAutomationSecurityForceDisable;
            }
            catch { /* not fatal on some installs */ }

            // sanity
            if (_excel == null)
                throw new InvalidOperationException("Failed to create Excel.Application.");
        }

        /// <summary>Open a workbook.</summary>
        public void OpenWorkbook(string filename, bool readOnly = true)
        {
            if (string.IsNullOrWhiteSpace(filename))
                throw new ArgumentException("Filename cannot be null or empty.", nameof(filename));

            if (!System.IO.File.Exists(filename))
                throw new System.IO.FileNotFoundException($"Excel file not found at path: {filename}", filename);

            if (_excel is null)
                throw new InvalidOperationException("Excel application is not initialized. Call InitializeAsync() first.");

            TryLog($"Opening Excel workbook: {filename} (ReadOnly={readOnly})");

            // Named args avoid the large parameter list on COM method
            _workBook = _excel.Workbooks.Open(
                Filename: filename,
                ReadOnly: readOnly
            );
        }

        /// <summary>Save the active workbook (same format).</summary>
        public void Save()
        {
            if (_workBook is null) throw new InvalidOperationException("No workbook is open.");
            TryLog("Saving workbook...");
            _workBook.Save();
        }

        /// <summary>Save As a new path (same format).</summary>
        public void SaveAs(string filename)
        {
            if (string.IsNullOrWhiteSpace(filename))
                throw new ArgumentException("Filename cannot be null or empty.", nameof(filename));
            if (_workBook is null) throw new InvalidOperationException("No workbook is open.");

            TryLog($"Saving workbook as: {filename}");
            _workBook.SaveAs(Filename: filename);
        }

        /// <summary>Export as plain text (*.txt, tab-delimited).</summary>
        public void SaveAsText(string filename)
        {
            if (string.IsNullOrWhiteSpace(filename))
                throw new ArgumentException("Filename cannot be null or empty.", nameof(filename));
            if (_workBook is null) throw new InvalidOperationException("No workbook is open.");

            TryLog($"Exporting workbook to text: {filename}");

            _workBook.SaveAs(
                Filename: filename,
                FileFormat: Excel.XlFileFormat.xlTextWindows,
                AccessMode: Excel.XlSaveAsAccessMode.xlNoChange
            );
        }

        /// <summary>Close the workbook (without saving changes).</summary>
        public void CloseWorkbook()
        {
            TryLog("Closing workbook...");
            try { _workBook?.Close(SaveChanges: false); }
            finally { ReleaseCom(ref _workBook); }
        }

        /// <summary>Dispose Excel and release COM.</summary>
        public void Dispose()
        {
            TryLog("Disposing Excel objects...");

            try
            {
                // Close WB if still open
                try { _workBook?.Close(SaveChanges: false); } catch { /* ignore */ }
                ReleaseCom(ref _workBook);

                // Quit Excel
                try { _excel?.Quit(); } catch { /* ignore */ }
                ReleaseCom(ref _excel);

                // Encourage COM cleanup
                GC.Collect();
                GC.WaitForPendingFinalizers();
                GC.Collect();
                GC.WaitForPendingFinalizers();
            }
            catch { /* best effort */ }
        }

        // -------- helpers --------

        private void TryLog(string message)
        {
            if (!string.IsNullOrWhiteSpace(message))
            {
                if (_logger != null) _logger.LogInformation(message);
                else Console.WriteLine($"[{DateTime.Now:HH:mm:ss}] {message}");
            }
        }

        private static void ReleaseCom<T>(ref T? comObj) where T : class
        {
            if (comObj != null && Marshal.IsComObject(comObj))
            {
                try { Marshal.FinalReleaseComObject(comObj); }
                catch { /* ignore */ }
            }
            comObj = null;
        }
    }
}
