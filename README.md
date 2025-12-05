USE [Quality_Control];
GO

SET ANSI_NULLS ON;
GO
SET QUOTED_IDENTIFIER ON;
GO

/********************************************************************
 Author:        Vivek Reddy Mandadi
 Create date:   2025-12-05
 Description:   When specific MCR_GridTracker columns change,
                set IsUpdateCompletedinAutoQC = 0.
*********************************************************************/
CREATE TRIGGER [dbo].[trg_MCR_GridTracker_Update_AutoQC]
ON [dbo].[MCR_GridTracker]
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    -- Fire ONLY when any of these columns are updated
    IF UPDATE(CopayGridStatus)
       OR UPDATE(GridEffectiveDate)
       OR UPDATE(PlanYear)
       OR UPDATE(AccumPlatform)
       OR UPDATE(Comments)
    BEGIN
        UPDATE GT
        SET IsUpdateCompletedinAutoQC = 0
        FROM dbo.MCR_GridTracker GT
        INNER JOIN inserted i ON GT.ID = i.ID;
    END
END;
GO
