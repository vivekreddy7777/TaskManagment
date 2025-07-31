Here is the fully detailed, **professionally formatted use case documentation** for `BTE_Export_Job`, ready for **Confluence or internal team use**. This version includes **interactive + service mode**, `Program.cs` behavior, `clsExport` logic, ZCLS-specific paths, and all stored procedures involved.

---

## 🧾 **Service Name**

**BTE-Export-Job**

---

## ✅ **Use Case Approver**

Steven

---

## ✅ **Approved (Yes/No)**

**No**

---

## 💬 **Comments**

Awaiting validation for ZCLS file generation and production path mapping for distributed file drops.

---

## 📌 **Use Case – 1: Export Request via Framework or Manual Execution**

| **Input**      | **Output**                      | **Results (Framework vs .NET 8)**       | **Comments**                             |
| -------------- | ------------------------------- | --------------------------------------- | ---------------------------------------- |
| `ExportJob_ID` | Excel/Text file (saved to disk) | `RowCount`, `JobStatus`, `Completed_TS` | Supports NDC, QC, ZCLS, Clinical exports |

---

## 🔁 **Supported Execution Modes**

| **Mode**         | **Description**                                                              |
| ---------------- | ---------------------------------------------------------------------------- |
| **Interactive**  | Developer runs export manually via console or tool passing `ExportJob_ID`.   |
| **Service Mode** | .NET background job polls for pending export jobs from DB every few minutes. |

---

## 🔄 **Steps Performed by `BTE_Export_Job`**

---

### **1. Get Job Details**

* Pulls metadata from `BTE_Export_Main_Request` using:

  * `procBTE_Export_MainRequest_SELECT`
* Populates fields:

  * `ExportType`, `TemplateName`, `Requestor`, `Email`, `Environment`

---

### **2. Validate Export Type**

* Based on `ExportType`, dynamically select stored procedure:

| **ExportType** | **Procedure Called**        |
| -------------- | --------------------------- |
| `NDC`          | `procExport_NDC`            |
| `QC`           | `procExport_QC_Validation`  |
| `ZCLS`         | `procExport_ZCLS_Report`    |
| `Clinical`     | `procExport_ClinicalReview` |

* Mapping is fetched via config, dictionary, or DB table.

---

### **3. Set File Path and Export File Name**

* Formats:

  * `FileName = "{ExportType}_export_{ExportJob_ID}_{Timestamp}.xlsx"`
* Saves path using `ExportRootPath` or template folder defined in settings.
* Validates:

  * Network access
  * Folder permissions
  * Path existence

---

### **4. Call Stored Procedure for Export Data**

* Invokes SQL stored procedure with parameters like:

  * `@ExportJob_ID`
  * `@Filter_OrgID`, `@Carrier`, etc. (for ZCLS/QC)
* Captures:

  * `DataTable` / `DataReader`
  * `RowCount`

---

### **5. Generate Export File (Handled by `clsExport`)**

| **Export Type** | **Engine Used**             | **Output Format** |
| --------------- | --------------------------- | ----------------- |
| Excel           | OpenXML SDK via `clsExport` | `.xlsx`           |
| Text            | StreamWriter                | `.txt` (pipe/tab) |

#### 🔧 OpenXML Export Features:

* Shared string table optimization
* Header + data row mapping
* Auto column width calculation
* Optional tabs (copied from templates)
* VML comments and legacy drawing insertion
* Pivot table part adjustment (if present)
* Style, font, and border deduplication logic

#### 🔧 Text Export:

* Header + values separated by `|` or `\t`
* Supports large flat exports with minimal memory usage

---

### **6. Insert Comments (if required)**

* Comment text comes from SQL field (e.g., `Note`, `ReviewReason`)
* Logic:

  * Insert into `CommentsPart`
  * Anchor calculated using row/column position
  * Auto-size bounding box using pixel length of string

---

### **7. Email Notification (`SendEmail()`)**

| **Condition**       | **Email Type**  |
| ------------------- | --------------- |
| Export success      | `Success`       |
| Empty resultset     | `Empty Results` |
| Exception thrown    | `Error`         |
| Manual cancellation | `Cancellation`  |
| Fatal runtime error | `Fatal Error`   |

* Emails sent using SMTP
* HTML templates with export links, job metadata, and error (if any)

---

### **8. Update DB Status**

On Success:

```sql
UPDATE BTE_Export_Main_Request
SET JobStatus = 'SUCCESS',
    Completed_TS = GETDATE(),
    RowCount = @RowCount,
    Export_FileName = @FilePath
WHERE ExportJob_ID = @ExportJob_ID
```

On Failure:

```sql
UPDATE BTE_Export_Main_Request
SET JobStatus = 'FAILURE',
    Error_Description = @ExceptionMessage
WHERE ExportJob_ID = @ExportJob_ID
```

---

## 📊 **ZCLS Export Use Case (Detailed)**

### When `ExportType = 'ZCLS'`:

1. Call `procZCLS_Report_Input` to retrieve filters
2. Call `procExport_ZCLS_Report` with job ID and filters
3. Export to:

   ```
   \\Export\ZCLS\zcls_export_{ExportJob_ID}.xlsx
   ```
4. Generate file using OpenXML
5. Update DB with file name and row count

---

## 🖥️ **Program.cs Logic**

```csharp
static async Task Main(string[] args)
{
    int jobId = args.Length > 0 ? int.Parse(args[0]) : PromptUserForJobId();
    var service = new ExportJobService();

    var job = await service.GetExportJobDetails(jobId);

    switch (job.ExportType.ToUpperInvariant())
    {
        case "ZCLS":
            await service.ProcessZclsExport(job);
            break;
        case "NDC":
            await service.ProcessNdcExport(job);
            break;
        default:
            await service.ProcessGenericExport(job);
            break;
    }

    Console.WriteLine("Export job completed.");
}
```

---

## 📁 **File Output Locations**

| **Type** | **Path Format**                                 |
| -------- | ----------------------------------------------- |
| NDC      | `\\Export\NDC\ndc_export_{ExportJob_ID}.xlsx`   |
| QC       | `\\Export\QC\qc_export_{ExportJob_ID}.xlsx`     |
| ZCLS     | `\\Export\ZCLS\zcls_export_{ExportJob_ID}.xlsx` |
| Text     | `\\Export\Text\export_{ExportJob_ID}.txt`       |

---

## 📋 **Procedures Used**

| **Procedure**                       | **Purpose**                             |
| ----------------------------------- | --------------------------------------- |
| `procBTE_Export_MainRequest_SELECT` | Get job config and metadata             |
| `procExport_ZCLS_Report`            | Core export logic for ZCLS jobs         |
| `procExport_QC_Validation`          | Fetches QC validation export data       |
| `procExport_NDC`                    | Generates NDC report                    |
| `procExport_ClinicalReview`         | Clinical export variant                 |
| `procBTE_Export_MainRequest_UPDATE` | Marks export as complete or failed      |
| `procZCLS_Report_Input`             | Filters for OrgID, Carrier, GridVersion |

---

## 🧪 **Failure Scenarios**

| **Scenario**            | **System Behavior**                                           |
| ----------------------- | ------------------------------------------------------------- |
| Template not found      | Logs and sets job status to `FAILURE`                         |
| SQL error during export | Catches exception, sets `Error_Description`                   |
| No data returned        | Sends "Empty Results" email, status still marked as `SUCCESS` |
| File path invalid       | Aborts and logs exception                                     |
| OpenXML exception       | Logs and continues if non-fatal                               |

---

## 📂 **Framework Config Values**

| **Key**              | **Description**                          |
| -------------------- | ---------------------------------------- |
| `MainRequestTable`   | SQL table holding job metadata           |
| `ExportRootPath`     | Folder for generated file storage        |
| `ExportTemplatePath` | Folder where templates are located       |
| `EmailTemplatePath`  | HTML templates for email types           |
| `QueueTimeout`       | Duration in seconds for polling next job |

---
