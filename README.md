private static string ParamVal(object v) =>
    (v == null || v == DBNull.Value) ? "<NULL>" : v.ToString();

private static void DumpParams(ILogger log, IDbCommand cmd, string when)
{
    var sb = new StringBuilder();
    sb.AppendLine($"--- SQL PARAM DUMP ({when}) ---");
    foreach (IDataParameter p in cmd.Parameters)
        sb.AppendLine($"{p.ParameterName,-18} | {p.Direction,-6} | {p.DbType,-10} | Size={p.Size,-6} | Value={ParamVal(p.Value)}");
    log.LogInformation(sb.ToString());
}

DumpParams(_logger, cmd, "BEFORE");
