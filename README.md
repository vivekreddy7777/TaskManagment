string valuesRows = string.Join(", ", ids.Select(_ => "(CAST(? AS CHAR(32)))"));

// 2) Use TABLE(VALUES ...) and give the column name
//    IMPORTANT: note the dot after {rl} → DB0{rl}.ELG0.ELGSMK00
string db2Sql = $@"
SELECT
  RTRIM(E.MEMBERSHIP_NBR)                  AS MEMBER_ID,
  RTRIM(E.PERSON_NBR)                      AS PERSON_NBR,
  COALESCE(CHAR(E.TRANS_INDIV_AGN_ID), '') AS RVRSD_PTNT_ACN
FROM DB0{rl}.ELG0.ELGSMK00 E
JOIN TABLE(VALUES {valuesRows}) AS IDS(MEMBERSHIP_NBR)
  ON RTRIM(E.MEMBERSHIP_NBR) = RTRIM(IDS.MEMBERSHIP_NBR)
FOR FETCH ONLY WITH UR";p
