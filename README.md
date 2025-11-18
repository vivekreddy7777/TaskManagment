using System;
using System.Data.Common;
using System.Data.SqlClient;
using System.IO;

namespace BTE_Export_Service
{
    public static class CompareToolErrorHelper
    {
        public enum ErrorBucket
        {
            DatabaseIssue,
            DataIssue,
            ServerOrNetworkIssue,
            UnknownIssue
        }

        public enum ErrorSubType
        {
            ConnectivityDatabaseUnreachable,
            SqlTimeout,
            MissingObject,
            DeadlockConcurrency,
            DataFormatConversion,
            ConstraintConsistency,
            DiskSpaceResource,
            FileLocked,
            NetworkPathOrMove,
            TemplateMissing,
            InternalProcessing,
            Unknown
        }

        public class ErrorClassification
        {
            public ErrorBucket Bucket { get; private set; }
            public ErrorSubType SubType { get; private set; }

            public ErrorClassification(ErrorBucket bucket, ErrorSubType subType)
            {
                Bucket = bucket;
                SubType = subType;
            }

            public override string ToString()
            {
                return "Bucket=" + Bucket + "; SubType=" + SubType;
            }
        }

        // MAIN ENTRY
        public static ErrorClassification Classify(Exception ex)
        {
            if (ex is SqlException)
                return ClassifySqlException((SqlException)ex);

            if (ex is DbException)
                return new ErrorClassification(ErrorBucket.DatabaseIssue, ErrorSubType.SqlTimeout);

            if (ex is IOException)
                return ClassifyIOException((IOException)ex);

            string msg = (ex.Message ?? "").ToLower();

            if (msg.Contains("template") && msg.Contains("not found"))
                return new ErrorClassification(ErrorBucket.ServerOrNetworkIssue, ErrorSubType.TemplateMissing);

            if (msg.Contains("index was outside the bounds"))
                return new ErrorClassification(ErrorBucket.UnknownIssue, ErrorSubType.InternalProcessing);

            return new ErrorClassification(ErrorBucket.UnknownIssue, ErrorSubType.Unknown);
        }


        // USER MESSAGE BUILDER
        public static string BuildUserMessage(ErrorClassification c, long requestNumber)
        {
            string baseText;

            switch (c.Bucket)
            {
                case ErrorBucket.DatabaseIssue:
                    baseText = "Your request could not be completed due to a database issue. Please resubmit your request or open a Tech Triage ticket with your request number, and the following message: ";
                    break;

                case ErrorBucket.DataIssue:
                    baseText = "Your request could not be completed due to an unexpected data issue. Please resubmit your request or open a Tech Triage ticket with your request number, and the following message: ";
                    break;

                case ErrorBucket.ServerOrNetworkIssue:
                    baseText = "Your request could not be completed due to a server or network issue. Please resubmit your request or open a Tech Triage ticket with your request number, and the following message: ";
                    break;

                default:
                    baseText = "Your request could not be completed due to an unknown issue. Please resubmit your request or open a Tech Triage ticket with your request number, and the following message: ";
                    break;
            }

            return baseText + GetSubTypeLabel(c.SubType);
        }

        public static string BuildDeveloperContext(ErrorClassification c, long requestNumber)
        {
            return "Request #" + requestNumber + "; " + c.ToString();
        }

        // SUBTYPE LABEL
        private static string GetSubTypeLabel(ErrorSubType s)
        {
            switch (s)
            {
                case ErrorSubType.ConnectivityDatabaseUnreachable: return "Connectivity / Database Unreachable";
                case ErrorSubType.SqlTimeout: return "SQL Timeout";
                case ErrorSubType.MissingObject: return "Missing Table / Stored Procedure / Object";
                case ErrorSubType.DeadlockConcurrency: return "Deadlock / Concurrency Issue";
                case ErrorSubType.DataFormatConversion: return "Data Format / Conversion Issue";
                case ErrorSubType.ConstraintConsistency: return "Constraint or Data Consistency Issue";
                case ErrorSubType.DiskSpaceResource: return "Disk Space / Server Resource Issue";
                case ErrorSubType.FileLocked: return "File In Use / Locked File";
                case ErrorSubType.NetworkPathOrMove: return "Network Path / File Move Error";
                case ErrorSubType.TemplateMissing: return "Template Missing";
                case ErrorSubType.InternalProcessing: return "Internal Processing Error";
                default: return "Unknown Issue";
            }
        }


        // SQL CLASSIFIER
        private static ErrorClassification ClassifySqlException(SqlException ex)
        {
            string msg = (ex.Message ?? "").ToLower();

            if (msg.Contains("could not open a connection") || msg.Contains("network"))
                return new ErrorClassification(ErrorBucket.DatabaseIssue, ErrorSubType.ConnectivityDatabaseUnreachable);

            if (msg.Contains("timeout"))
                return new ErrorClassification(ErrorBucket.DatabaseIssue, ErrorSubType.SqlTimeout);

            if (msg.Contains("could not find stored procedure") ||
                msg.Contains("invalid object name"))
                return new ErrorClassification(ErrorBucket.DatabaseIssue, ErrorSubType.MissingObject);

            if (msg.Contains("deadlock"))
                return new ErrorClassification(ErrorBucket.DatabaseIssue, ErrorSubType.DeadlockConcurrency);

            if (msg.Contains("conversion failed") || msg.Contains("overflow"))
                return new ErrorClassification(ErrorBucket.DataIssue, ErrorSubType.DataFormatConversion);

            if (msg.Contains("conflicted with") && msg.Contains("constraint"))
                return new ErrorClassification(ErrorBucket.DataIssue, ErrorSubType.ConstraintConsistency);

            return new ErrorClassification(ErrorBucket.UnknownIssue, ErrorSubType.Unknown);
        }


        // IO CLASSIFIER
        private static ErrorClassification ClassifyIOException(IOException ex)
        {
            string msg = (ex.Message ?? "").ToLower();

            if (msg.Contains("disk"))
                return new ErrorClassification(ErrorBucket.ServerOrNetworkIssue, ErrorSubType.DiskSpaceResource);

            if (msg.Contains("being used by another process") || msg.Contains("cannot access the file"))
                return new ErrorClassification(ErrorBucket.ServerOrNetworkIssue, ErrorSubType.FileLocked);

            if (msg.Contains("network"))
                return new ErrorClassification(ErrorBucket.ServerOrNetworkIssue, ErrorSubType.NetworkPathOrMove);

            return new ErrorClassification(ErrorBucket.ServerOrNetworkIssue, ErrorSubType.Unknown);
        }
    }
}

--------------

using System;
using log4net;

namespace BTE_Export_Service
{
    public class BteSystemEmailSender
    {
        private readonly ILog _log;

        public BteSystemEmailSender(ILog log)
        {
            _log = log;
        }

        public void SendErrorEmailForException(ExportEmailContext ctx, Exception ex)
        {
            var classification = CompareToolErrorHelper.Classify(ex);

            string friendlyMessage =
                CompareToolErrorHelper.BuildUserMessage(classification, ctx.RequestNumber);

            string body = BuildErrorEmailBody(ctx, friendlyMessage);

            string devContext =
                CompareToolErrorHelper.BuildDeveloperContext(classification, ctx.RequestNumber);

            _log.Error("Export failed. " + devContext, ex);

            QueueSystemEmail(ctx,
                "BTA Export Service Error (Request # " + ctx.RequestNumber + ")",
                body);
        }


        private string BuildErrorEmailBody(ExportEmailContext ctx, string friendlyMessage)
        {
            string template =
@"<style>td{{text-align:right;vertical-align:top;padding-right:10;font-weight:bold;white-space:nowrap;}}
.tdd{{text-align:left;vertical-align:top;}}</style>
<div style='font-family:Calibri;font-size:11pt;'>
    <span style='font-weight:bold;'>***IMPORTANT NOTICE***</span><br/><br/>
    A BTA Export and Reporting file has failed, details are below:<br/><br/>
    <table border='0' cellspacing='0' cellpadding='0'>
        <tr><td>Request #:</td><td class='tdd'>{0}</td></tr>
        <tr><td>Request Date:</td><td class='tdd'>{1}</td></tr>
        <tr><td>Report Name:</td><td class='tdd'>{2}</td></tr>
        <tr><td>Requestor Email:</td><td class='tdd'>{3}</td></tr>
        <tr><td>Requestor LAN ID:</td><td class='tdd'>{4}</td></tr>
        <tr><td>Error Details:</td><td class='tdd'>{5}</td></tr>
    </table>
    <br/>
    Thank you for using the Export and Reporting Service provided by Q-Core.<br/>
</div>";

            return string.Format(template,
                ctx.RequestNumber.ToString(),
                ctx.RequestedDate.ToString(),
                ctx.ReportName,
                ctx.RequestorEmail,
                ctx.Requestor.Replace("\"", ""),
                friendlyMessage);
        }


        private void QueueSystemEmail(ExportEmailContext ctx, string subject, string body)
        {
            SqlUtilities.ProcCommand(
                "procBTE_System_Emails_INSERT",
                new Library
                {
                    { "@MailTo", "Q-CoreExpress-Scripts.com" },
                    { "@MailCC", ctx.ReportOwnerEmail },
                    { "@MailSubject", subject },
                    { "@MailBody", body },
                    { "@Procedure", ctx.ProcedureName }
                })
            .Run();
        }
    }
}



catch (Exception ex)
{
    var ctx = new ExportEmailContext
    {
        RequestNumber     = request.Request_Number,
        RequestedDate     = request.Requested_Date,
        ReportName        = request.Report_Name,
        Requestor         = request.Requestor,
        RequestorEmail    = request.Requestor_Email,
        ReportOwnerEmail  = request.Report_Owner_Email,
        ProcedureName     = "BTE_Export_Job"
    };

    var emailSender = new BteSystemEmailSender(_log);
    emailSender.SendErrorEmailForException(ctx, ex);

    throw;
}
