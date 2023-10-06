```sql
declare @p28 [uniqueidentifier] = '9350741a-3f1c-4a56-aa6a-cb94bd1f30ca', 
@p29 [uniqueidentifier] = '71b3e352-be46-4ec9-9c75-7b60b3f43828', 
@p30 [uniqueidentifier] = '2e8885c8-547b-42a7-ad15-c221e4242c53', 
@p32 [int] = 0, 
@p35 [int] = 1, 
@p36 [varchar](11) = 'B01.047.005', 
@p38 [datetime] = '2023-09-01 00:00:00.000', 
@p40 [bit] = 1, 
@p41 [bit] = 0, 
@p42 [uniqueidentifier] = '077c8a28-ef23-41ad-a056-8f8bab4a6c61', 
@p47 [int] = 2, 
@p49 [int] = 12601, 
@p50 [uniqueidentifier] = '05351f94-95ba-4139-b77c-563ac5b62f01', 
@p51 [uniqueidentifier] = 'f6619585-cbd3-4a01-bbd3-101e3701d442', 
@p52 [uniqueidentifier] = 'e27d1f3f-ade4-4a19-82d5-14b82a376d75', 
@p53 [uniqueidentifier] = '781d9ab7-71c6-4708-96da-734236fff03f' 

SELECT 
  t0.ExaminationTypeLinkID, 
  t0.ExaminationBase, 
  t2.type_, 
  t1.Post, 
  t6.type_, 
  t6.Division, 
  t7.Branch, 
  t11.LpuMain, 
  t12.Name, 
  t2.EventDate, 
  t0.Mkb, 
  t0.ExecutionDate, 
  t0.MedicalService, 
  t27.type_, 
  t27.Code, 
  t27.Name 
FROM 
  AKUZ.T_EXAMINATION_TYPE_LINK t0 
  LEFT JOIN AKUZ.T_EXAMINATION_BASE t1 ON t1.EventID = t0.ExaminationBase 
  LEFT JOIN AKUZ.T_EVENT_BASE t2 ON t2.EventID = t0.ExaminationBase 
  and t2.type_ in (552, 553, 554, 58270, 551) 
  LEFT JOIN AKUZ.T_AMBULANCE_CARD t3 ON t3.AmbulanceCardID = t2.AmbulanceCard 
  LEFT JOIN AKUZ.T_LPU_MAIN t4 ON t4.LpuID = t3.CardLpu 
  LEFT JOIN AKUZ.T_POST t5 ON t5.ResourceID = t1.Post 
  LEFT JOIN AKUZ.T_RESOURCE t6 ON t6.ResourceID = t1.Post 
  and t6.type_ in (501) 
  LEFT JOIN AKUZ.T_DIVISION t7 ON t7.DivisionID = t6.Division 
  LEFT JOIN AKUZ.T_BRANCH t11 ON t11.LpuID = t7.Branch 
  LEFT JOIN AKUZ.T_LPU_MAIN t12 ON t12.LpuID = t11.LpuMain 
  LEFT JOIN AKUZ.T_MEDICAL_SERVICE t27 ON t27.MedicalServiceID = t0.MedicalService 
WHERE 
  (
    (
      t1.ExaminationType = @p28 
      OR t1.ExaminationType = @p29 
      OR t1.ExaminationType = @p30
    ) 
    AND NOT EXISTS (
      SELECT 
        * 
      FROM 
        AKUZ.T_EXAMINATION_TYPE_LINK t31 
      WHERE 
        t1.EventID = t31.ExaminationBase 
        AND t31.Status = @p32
    ) 
    AND EXISTS (
      SELECT 
        * 
      FROM 
        AKUZ.T_EXAMINATION_TYPE_LINK t33 
        LEFT JOIN AKUZ.T_MEDICAL_SERVICE t34 ON t34.MedicalServiceID = t33.MedicalService 
      WHERE 
        t1.EventID = t33.ExaminationBase 
        AND t33.Status = @p35 
        AND t34.Code = @p36
    ) 
    AND cast(
      (
        SELECT 
          MAX(t37.ExecutionDate) 
        FROM 
          AKUZ.T_EXAMINATION_TYPE_LINK t37 
        WHERE 
          t1.EventID = t37.ExaminationBase 
          AND t37.Status = @p35
      ) as [datetime]
    )>= @p38 
    AND cast(
      (
        SELECT 
          MAX(t39.ExecutionDate) 
        FROM 
          AKUZ.T_EXAMINATION_TYPE_LINK t39 
        WHERE 
          t1.EventID = t39.ExaminationBase 
          AND t39.Status = @p35
      ) as [datetime]
    )<= @p38 
    AND (
      (
        @p40 = @p41 
        AND t4.LpuID = @p42 
        AND EXISTS (
          SELECT 
            * 
          FROM 
            AKUZ.T_REGISTRATION t43 
            LEFT JOIN AKUZ.T_LPU_MAIN t44 ON t44.LpuID = t43.LpuMain 
          WHERE 
            t44.LpuID = @p42 
            AND t43.Patient = t3.Patient 
            AND t43.Status = @p35 
            AND t43.RegistrationID =(
              SELECT 
                TOP 1 t45.RegistrationID 
              FROM 
                AKUZ.T_REGISTRATION t45 
                LEFT JOIN AKUZ.T_REGISTRATION_CANCEL t46 ON t46.Registration = t45.RegistrationID 
              WHERE 
                (
                  t45.Patient = t3.Patient 
                  AND t45.RegistrationDate <= @p38 
                  AND (
                    t46.RegistrationCancelID IS NULL 
                    OR NOT t46.Status = @p47
                  ) 
                  AND t45.AttachmentStatus > @p32 
                  AND (
                    t45.[State] IS NULL 
                    OR 1 = 1
                  )
                ) 
              ORDER BY 
                t45.RegistrationDate DESC
            )
        )
      ) 
      OR (
        @p41 = @p41 
        AND t4.LpuID = @p42
      )
    ) 
    AND (
      t7.Branch IS NULL 
      OR NULL IS NULL
    ) 
    AND NOT t2.CloseDate IS NULL 
    AND DATEPART(Year, t2.CloseDate)= DATEPART(Year, @p38) 
    AND EXISTS (
      SELECT 
        * 
      FROM 
        AKUZ.T_VISIT t48 
      WHERE 
        t48.type_ IN (580, 505, 558, 581, 647, 545, 664) 
        AND t1.EventID = t48.AmbulanceEvent 
        AND t48.PaymentType = @p49 
        AND (
          t48.VisitPurpose = @p50 
          OR t48.VisitPurpose = @p51 
          OR t48.VisitPurpose = @p52 
          OR t48.VisitPurpose = @p53
        ) 
        AND NOT t48.OutcomeRecord IS NULL
    ) 
    AND (
      (
        EXISTS (
          SELECT 
            * 
          FROM 
            AKUZ.T_SCHET_SLUCH_116 t54 
            LEFT JOIN AKUZ.T_SCHET_SLUCH t55 ON t55.SchetSluchID = t54.SchetSluchID 
          WHERE 
            t55.Event = t1.EventID
        ) 
        AND @p41 = @p40
      ) 
      OR @p40 = @p40
    )
  ) 
ORDER BY 
  t0.ExecutionDate DESC
```
