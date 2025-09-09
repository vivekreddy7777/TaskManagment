BEGIN CATCH
    DECLARE @ThisProc SYSNAME = QUOTENAME(OBJECT_SCHEMA_NAME(@@PROCID)) + '.' + QUOTENAME(OBJECT_NAME(@@PROCID));
    DECLARE @RunId UNIQUEIDENTIFIER = NEWID();

    DECLARE @ErrMsg NVARCHAR(MAX) = CONCAT(
        '[ORIGIN=', @ThisProc, '][RUNID=', @RunId, '] ',
        'Table=', ISNULL(@TempTableName,'(null)'), 
        ' | Error=', ERROR_NUMBER(),
        ', Sev=', ERROR_SEVERITY(),
        ', State=', ERROR_STATE(),
        ', Line=', ERROR_LINE(),
        ' | ', ERROR_MESSAGE()
    );

    SET @ProcResult = @ErrMsg;

    IF @Debug = 1 PRINT @ErrMsg;

    THROW 51010, @ErrMsg, 1;
END CATCH
