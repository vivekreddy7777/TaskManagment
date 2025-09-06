private string GetEffectiveTableName()
{
    if (_gridType == 1)      return _valTableName;   // Validation path
    else if (_gridType == 4) return _tempTableName;  // BTE path
    else                     return _tableName;      // Normal compare path
}
