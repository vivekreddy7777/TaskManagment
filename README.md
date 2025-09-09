BEGIN TRY
    IF @Debug=1 PRINT CONCAT('[',@ThisProc,'][',@RunId,'][STEP=Validation_BulkInsert] start');
    EXEC dbo.procCompareTool_Validation_BulkInsert
         @Password=@Password,
         @TableName=@ValTableName,
         @TempTableName=@ValTempTableName,
         @FileName=@FileName,
         @ProcResult=@ProcResult OUTPUT,
         @Debug=@Debug;
    IF @Debug=1 PRINT CONCAT('[',@ThisProc,'][',@RunId,'][STEP=Validation_BulkInsert] done: ', @ProcResult);
END TRY
BEGIN CATCH
    -- bubble with a precise tag so you know *which* child failed
    THROW 51010, CONCAT(
        '[ORIGIN=',@ThisProc,'][RUNID=',@RunId,'][STEP=Validation_BulkInsert] ',
        'Err=',ERROR_NUMBER(),', Sev=',ERROR_SEVERITY(),', State=',ERROR_STATE(),
        ', Line=',ERROR_LINE(),' | ',ERROR_MESSAGE()
    ), 1;
END CATCH;
