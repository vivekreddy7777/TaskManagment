// Assume these are already retrieved from CyberArk
string sqlUser = runtimeContext.SqlUser;
string sqlPassword = runtimeContext.SqlPassword;

string db2User = runtimeContext.Db2User;
string db2Password = runtimeContext.Db2Password;

string teradataUser = runtimeContext.TeradataUser;
string teradataPassword = runtimeContext.TeradataPassword;


// ======================
// SQL SERVER
// ======================

// Base SQL connection string from config server (NO credentials)
string sqlBaseConnString = mergedConfig["QCoreDBConnectionString"];

// Build SQL connection string with CyberArk creds
var sqlBuilder = new SqlConnectionStringBuilder(sqlBaseConnString)
{
    UserID = sqlUser,
    Password = sqlPassword,
    IntegratedSecurity = false,
    TrustServerCertificate = true,
    Encrypt = true
};

string sqlConnectionString = sqlBuilder.ConnectionString;


// ======================
// DB2
// ======================

// Values from config server
string db2Server = mergedConfig["Db2Server"];
string db2Port = mergedConfig["Db2Port"];
string db2Database = mergedConfig["Db2Database"];

// Build DB2 connection string
string db2ConnectionString =
    $"Server={db2Server}:{db2Port};" +
    $"Database={db2Database};" +
    $"UID={db2User};" +
    $"PWD={db2Password};";


// ======================
// TERADATA
// ======================

// Value from config server
string teradataServer = mergedConfig["TeradataServer"];

// Build Teradata connection string
string teradataConnectionString =
    $"Data Source={teradataServer};" +
    $"User ID={teradataUser};" +
    $"Password={teradataPassword};";


// ======================
// OPTIONAL: STORE IN RUNTIME CONTEXT
// ======================

runtimeContext.SqlConnectionString = sqlConnectionString;
runtimeContext.Db2ConnectionString = db2ConnectionString;
runtimeContext.TeradataConnectionString = teradataConnectionString;


// ======================
// OPTIONAL: SAFE VERIFICATION (NO SECRETS)
// ======================

Log.Information("Connection strings built successfully using CyberArk credentials");
