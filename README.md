using System;
using System.Diagnostics;
using System.IO;
using System.Runtime.InteropServices;
using System.Text.RegularExpressions;
using Microsoft.Office.Interop.Excel;

namespace BTA
{
    /// <summary>
    /// Thin wrapper around Microsoft.Office.Interop.Excel for your service.
    /// No ILogger dependency. Logs via CurrentJob if available; otherwise Debug.WriteLine.
    /// </summary>
    public class ExcelObjects : IDisposable
    {
        #region Regex examples (keep if you still use them elsewhere)
        private static readonly Regex IgnoreFormats = new Regex(@"^([@]|General|[0-9\-\,\.\(\)\ ]+)$",
            RegexOptions.Compiled | RegexOptions.CultureInvariant);
        private static readonly Regex CurrencyFormat = new Regex(@"^(\$|£|€).{1,10}$",
            RegexOptions.Compiled | RegexOptions.CultureInvariant);
        #endregion

        /// <summary>Whatever your Job type is; used for optional logging.</summary>
        public object CurrentJob { get; }

        /// <summary>Excel application COM object.</summary>
        public Application Excel { get; private set; }

        /// <summary>Currently opened workbook COM object.</summary>
        public Workbook WorkBook { get; private set; }

        public ExcelObjects(object jobDetails = null)
        {
            CurrentJob = jobDetails;
        }

        #region Logging helpers (no ILogger required)
        private static void TryLogInfo(object job, string message)
        {
            if (job == null) { Debug.WriteLine(message); return; }
            try
            {
                // If your Job has LogInformation(string)
                var mi = job.GetType().GetMethod("LogInformation", new[] { typeof(string) });
                if (mi != null) { mi.Invoke(job, new object[] { message }); return; }
            }
            catch { /* ignore and fall back */ }
            Debug.WriteLine(message);
        }

        private static void TryLogError(object job, Exception ex, string message)
        {
            if (job == null) { Debug.WriteLine($"{message}: {ex}"); return; }
            try
            {
                // If your Job has LogError(Exception,string)
                var mi = job.GetType().GetMethod("LogError", new[] { typeof(Exception), typeof(string) });
                if (mi != null) { mi.Invoke(job, new object[] { ex, message }); return; }
            }
            catch { /* ignore and fall back */ }
            Debug.WriteLine($"{message}: {ex}");
        }
        #endregion

        /// <summary>
        /// Open Excel application (hidden), disable alerts/macros.
        /// </summary>
        public void Initialize()
        {
            TryLogInfo(CurrentJob, "Starting Excel application...");
            try
            {
                Excel = new Application
                {
                    Visible = false,
                    DisplayAlerts = false,
                    ScreenUpdating = false,
                    AskToUpdateLinks = false
                };

                // Disable macros
                try
                {
                    Excel.AutomationSecurity =
                        Microsoft.Office.Core.MsoAutomationSecurity.msoAutomationSecurityForceDisable;
                }
                catch { /* some environments don't expose AutomationSecurity */ }

                TryLogInfo(CurrentJob, "Excel application started.");
            }
            catch (Exception ex)
            {
                TryLogError(CurrentJob, ex, "Failed to start Excel");
                Dispose();
                throw;
            }
        }

        /// <summary>
        /// Load the specified file.
        /// </summary>
        public void OpenWorkbook(string filename, bool readOnly = true)
        {
            if (string.IsNullOrWhiteSpace(filename))
                throw new ArgumentException("Filename cannot be null or empty", nameof(filename));

            if (!File.Exists(filename))
                throw new FileNotFoundException($"Excel file not found at path: {filename}", filename);

            if (Excel == null) Initialize();

            try
            {
                TryLogInfo(CurrentJob, $"Opening Excel Workbook: {filename} (ReadOnly={readOnly})");

                // Use a simple Open overload; other args default.
                WorkBook = Excel.Workbooks.Open(
                    Filename: filename,
                    ReadOnly: readOnly,
                    Editable: false,
                    IgnoreReadOnlyRecommended: true
                );
            }
            catch (Exception ex)
            {
                TryLogError(CurrentJob, ex, $"Failed to open Excel workbook: {filename}");
                throw;
            }
        }

        /// <summary>Close current workbook (no save).</summary>
        public void CloseWorkbook()
        {
            try
            {
                WorkBook?.Close(SaveChanges: false);
            }
            catch (Exception ex)
            {
                TryLogError(CurrentJob, ex, "Error closing workbook");
            }
            finally
            {
                ReleaseCom(ref WorkBook);
            }
        }

        /// <summary>Save the active file with the given name (native Excel format).</summary>
        public void SaveAs(string filename)
        {
            if (WorkBook == null)
                throw new InvalidOperationException("No workbook is open to save.");

            if (string.IsNullOrWhiteSpace(filename))
                throw new ArgumentException("Filename cannot be null or empty", nameof(filename));

            try
            {
                TryLogInfo(CurrentJob, $"Saving Excel workbook to: {filename}");
                WorkBook.SaveAs(Filename: filename);
            }
            catch (Exception ex)
            {
                TryLogError(CurrentJob, ex, $"Failed to save Excel workbook: {filename}");
                throw;
            }
        }

        /// <summary>Save the active file as Windows text (*.txt).</summary>
        public void SaveAsText(string filename)
        {
            if (WorkBook == null)
                throw new InvalidOperationException("No workbook is open to save as text.");

            if (string.IsNullOrWhiteSpace(filename))
                throw new ArgumentException("Filename cannot be null or empty", nameof(filename));

            try
            {
                TryLogInfo(CurrentJob, $"Saving Excel workbook as text to: {filename}");
                WorkBook.SaveAs(
                    Filename: filename,
                    FileFormat: XlFileFormat.xlTextWindows,
                    AccessMode: XlSaveAsAccessMode.xlNoChange
                );
            }
            catch (Exception ex)
            {
                TryLogError(CurrentJob, ex, $"Failed to save Excel workbook as text: {filename}");
                throw;
            }
        }

        /// <summary>Dispose/cleanup COM objects safely.</summary>
        public void Dispose()
        {
            TryLogInfo(CurrentJob, "Disposing Excel objects...");

            // Close workbook
            try { WorkBook?.Close(SaveChanges: false); } catch { /* ignore */ }
            ReleaseCom(ref WorkBook);

            // Quit Excel app
            try { Excel?.Quit(); } catch { /* ignore */ }
            ReleaseCom(ref Excel);

            // Encourage COM cleanup
            try
            {
                GC.Collect();
                GC.WaitForPendingFinalizers();
                GC.Collect();
                GC.WaitForPendingFinalizers();
            }
            catch { /* ignore */ }

            TryLogInfo(CurrentJob, "Excel objects disposed.");
        }

        private static void ReleaseCom<T>(ref T comObject) where T : class
        {
            if (comObject == null) return;
            try
            {
                if (Marshal.IsComObject(comObject))
                    Marshal.FinalReleaseComObject(comObject);
            }
            catch { /* ignore */ }
            finally
            {
                comObject = null;
            }
        }
    }
}
