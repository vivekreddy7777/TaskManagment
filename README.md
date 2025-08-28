using System;
using System.Diagnostics;
using System.Runtime.InteropServices;
using Microsoft.Office.Interop.Excel; // COM interop (Microsoft.Office.Interop.Excel)
using ExcelApp = Microsoft.Office.Interop.Excel.Application;

namespace BTA
{
    /// <summary>
    /// Minimal, safe Excel COM wrapper used by CompareTool.
    /// </summary>
    public sealed class ExcelObjects : IDisposable
    {
        // Backing fields (so we can pass them by ref during COM release)
        private ExcelApp? _excel;
        private Workbook? _workBook;

        // Optional context for logging (your job object) and a simple logging hook
        public object? CurrentJob { get; }
        private readonly Action<string>? _log;

        // Read-only accessors if you still want to inspect them
        public ExcelApp? Excel => _excel;
        public Workbook? WorkBook => _workBook;

        public ExcelObjects(object? jobDetails = default, Action<string>? log = default)
        {
            CurrentJob = jobDetails;
            _log = log;
        }

        /// <summary>
        /// Open Excel application (hidden, no prompts, macros disabled).
        /// </summary>
        public void Initialize()
        {
            TryLog("Opening Excel application...");
            try
            {
                _excel = new ExcelApp
                {
                    Visible = false,
                    DisplayAlerts = false
                };

                // Disable macros (no security prompts)
                // ReSharper disable once SuspiciousTypeConversion.Global
                Excel.AutomationSecurity = Microsoft.Office.Core.MsoAutomationSecurity.msoAutomationSecurityForceDisable;

                TryLog("Excel application opened.");
            }
            catch
            {
                // Ensure we don’t leave a half-constructed COM object around
                Dispose();
                throw;
            }
        }

        /// <summary>
        /// Opens a workbook at the given path.
        /// </summary>
        public void OpenWorkbook(string filename, bool readOnly = true)
        {
            if (string.IsNullOrWhiteSpace(filename))
                throw new ArgumentException("Filename cannot be null or empty.", nameof(filename));

            if (!System.IO.File.Exists(filename))
                throw new System.IO.FileNotFoundException($"Excel file not found at path: {filename}", filename);

            if (_excel is null)
                throw new InvalidOperationException("Excel application is not initialized. Call Initialize() first.");

            TryLog($"Opening Excel workbook: {filename} (ReadOnly={readOnly})");

            // Use named args to avoid the giant Open signature
            _workBook = _excel.Workbooks.Open(
                Filename: filename,
                ReadOnly: readOnly
            );
        }

        /// <summary>
        /// Saves the active workbook to the given path (same format).
        /// </summary>
        public void SaveAs(string filename)
        {
            if (_workBook is null)
                throw new InvalidOperationException("No workbook is open.");

            TryLog($"Saving workbook as: {filename}");
            _workBook.SaveAs(Filename: filename);
        }

        /// <summary>
        /// Exports the active workbook to a tab-delimited text file.
        /// </summary>
        public void SaveAsText(string filename)
        {
            if (_workBook is null)
                throw new InvalidOperationException("No workbook is open.");

            TryLog($"Saving workbook as TEXT: {filename}");
            _workBook.SaveAs(
                Filename: filename,
                FileFormat: XlFileFormat.xlTextWindows,
                AccessMode: XlSaveAsAccessMode.xlNoChange
            );
        }

        /// <summary>
        /// Closes the current workbook (no save).
        /// </summary>
        public void CloseWorkbook()
        {
            try
            {
                _workBook?.Close(SaveChanges: false);
            }
            finally
            {
                ReleaseCom(ref _workBook);
            }
        }

        /// <summary>
        /// Full cleanup for Excel COM.
        /// </summary>
        public void Dispose()
        {
            TryLog("Disposing Excel objects...");

            // Close workbook first
            try { _workBook?.Close(SaveChanges: false); } catch { /* ignore */ }
            ReleaseCom(ref _workBook);

            // Quit Excel app
            try { _excel?.Quit(); } catch { /* ignore */ }
            ReleaseCom(ref _excel);

            // Encourage COM cleanup
            try
            {
                GC.Collect();
                GC.WaitForPendingFinalizers();
                GC.Collect();
                GC.WaitForPendingFinalizers();
            }
            catch
            {
                // Best effort
            }
        }

        // ----------------- helpers -----------------

        private void TryLog(string message)
        {
            try
            {
                // Prefer delegate if provided
                _log?.Invoke(message);

                // Fallback to Debug to ensure something is written even without DI logger
                Debug.WriteLine($"[ExcelObjects] {message}");
            }
            catch
            {
                // Never let logging break the flow
            }
        }

        private static void ReleaseCom<T>(ref T? com) where T : class
        {
            if (com is null) return;

            try
            {
                // Release RCW
                Marshal.FinalReleaseComObject(com);
            }
            catch
            {
                // ignore
            }
            finally
            {
                com = null;
            }
        }
    }
}
