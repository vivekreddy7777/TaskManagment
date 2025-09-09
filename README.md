BEGIN CATCH
    DECLARE 
        @ErrNum int        = ERROR_NUMBER(),
        @ErrSev int        = ERROR_SEVERITY(),
        @ErrState int      = ERROR_STATE(),
        @ErrLine int       = ERROR_LINE(),
        @ErrProc nvarchar(200) = ERROR_PROCEDURE(),
        @ErrMsg nvarchar(max)  = ERROR_MESSAGE();

    SET @ProcResult = CONCAT(
        'Validation: File Upload Error. ',
        'File: ', @FileName,
        ' | Table: ', @TempTableName,
        ' | Error ', @ErrNum,
        ', Severity ', @ErrSev,
        ', State ', @ErrState,
        ', Line ', @ErrLine,
        CASE WHEN @ErrProc IS NOT NULL THEN CONCAT(', Proc ', @ErrProc) ELSE '' END,
        '. Msg: ', @ErrMsg
    );

    IF @Debug = 1
    BEGIN
        PRINT @ProcResult;
        PRINT 'Command was: ' + @cmd;
    END
END CATCH
