using System;
using System.Data;
using System.Data.SqlClient;

namespace BTE_CompareTool_Service
{
    public class EmailSender
    {
        private readonly string _connectionString;

        public EmailSender(string connectionString)
        {
            _connectionString = connectionString;
        }

        /// <summary>
        /// Sends a CompareTool system email by calling procBTE_System_Emails_INSERT
        /// </summary>
        public void SendSystemEmail(
            string to,
            string cc,
            string subject,
            string body)
        {
            using (var conn = new SqlConnection(_connectionString))
            using (var cmd = new SqlCommand("procBTE_System_Emails_INSERT", conn))
            {
                cmd.CommandType = CommandType.StoredProcedure;

                cmd.Parameters.AddWithValue("@MailTo", to ?? "");
                cmd.Parameters.AddWithValue("@MailCC", cc ?? "");
                cmd.Parameters.AddWithValue("@MailSubject", subject ?? "");
                cmd.Parameters.AddWithValue("@EmailBody", body ?? "");
                cmd.Parameters.AddWithValue("@Procedure", "CompareTool"); // matches your system

                conn.Open();
                cmd.ExecuteNonQuery();
            }
        }

        /// <summary>
        /// Build and send the user-facing error email
        /// </summary>
        public void SendErrorEmail(long requestNumber, Exception ex)
        {
            var c = CompareToolErrorHelper.Classify(ex);

            string subject = "CompareTool Error (Request " + requestNumber + ")";
            string message = CompareToolErrorHelper.BuildUserMessage(c, requestNumber);

            // In your job this is Requestor_Email
            string to = GetRequesterEmail(requestNumber);

            // Generic support mailbox
            string cc = "Q-CoreExpress-Scripts.com";

            SendSystemEmail(
                to: to,
                cc: cc,
                subject: subject,
                body: message
            );
        }

        // You already have this logic in your job - placeholder for completeness
        private string GetRequesterEmail(long requestNumber)
        {
            // Replace with your existing method to fetch requester email
            return "user@domain.com";
        }
    }
}
