
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Data.SqlClient;
using Microsoft.EntityFrameworkCore;

namespace BTA_CompareTool_Console.Services
{
    public class SystemVariableService
    {
        private readonly CompareToolConsoleDbContext _dbContext;

        public SystemVariableService(CompareToolConsoleDbContext dbContext)
        {
            _dbContext = dbContext;
        }

        private static string? CleanInput(string? input)
        {
            if (string.IsNullOrWhiteSpace(input))
                return input;

            return input.Trim().Trim('\"', '“', '”', '\'', '‘', '’');
        }

        public async Task<string?> GetAsync(string variable, string module, CancellationToken ct = default)
        {
            var v = CleanInput(variable);
            var m = CleanInput(module);

            const string sql = "SELECT dbo.fn_SystemVariable(@var, @mod) AS Value";

            var result = await _dbContext.Database
                .SqlQuery<string>(
                    sql,
                    new SqlParameter("@var", v ?? (object)DBNull.Value),
                    new SqlParameter("@mod", m ?? (object)DBNull.Value))
                .FirstOrDefaultAsync(ct);

            return result;
        }
    }
}
