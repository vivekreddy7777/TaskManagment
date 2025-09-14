cmd.CommandText = "dbo.procCompareTool_ClaimExport_ALTERNATE";

var pWm = cmd.CreateParameter();
pWm.ParameterName = "@WM_ID";
pWm.DbType = DbType.String;                 // SP expects varchar(max)
pWm.Value = Compare_ID.ToString();
cmd.Parameters.Add(pWm);

var pType = cmd.CreateParameter();
pType.ParameterName = "@Type";              // must match SP param name
pType.DbType = DbType.String;
pType.Value = string.IsNullOrWhiteSpace(ExportRequestType) ? "1" : ExportRequestType;
cmd.Parameters.Add(pType);

var p1 = cmd.CreateParameter();
p1.ParameterName = "@Parm1";
p1.DbType = DbType.String;
p1.Value = DBNull.Value;                    // optional, defaults to NULL
cmd.Parameters.Add(p1);

var p2 = cmd.CreateParameter();
p2.ParameterName = "@Parm2";
p2.DbType = DbType.String;
p2.Value = DBNull.Value;                    // optional, defaults to NULL
cmd.Parameters.Add(p2);

foreach (DbParameter p in cmd.Parameters)
    _logger.LogDebug("Inner Param: {Name}={Value} ({Type})",
        p.ParameterName, p.Value ?? "NULL", p.DbType);

await cmd.ExecuteNonQueryAsync(token);

var summary = 
    $"EXEC dbo.procCompareTool_ClaimExport_ALTERNATE " +
    $"@WM_ID={Compare_ID}, @Type='{pType.Value}', @Parm1=NULL, @Parm2=NULL";

return ("CompareAlternate", summary);
