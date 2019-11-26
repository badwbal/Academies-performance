# Academies-performance

--This code calculates the variance of all pupils nationally for the reading, writing and progress measures respectively at KS2. These will be needed for the confidence intervals

Use Assessment
go
select var(READPROGSCORE_P) as [Reading progress score variance],
var(WRITPROGSCORE_P) as [Writing progress score variance],
var(MATPROGSCORE_P) as [Maths progress score variance]
into #nationalvariances
from [dbo].[KS2_2016_Pupil_3rdamended_anonymised]
Where NFTYPE in(21,22,23,24,25,20,51,52,57,58) and SCHRES=1 and ENDKS=1

/****************************************************************************/

--This snippet of code is used to produce contextual information in the absence of TELIG. TELIG will come when the results are final

Use Assessment
go
Select URN, CANDNO,SCHRES, ENDKS, KS1READPS_P, KS1WRITPS_P, KS1MATPS_P,
Case when schres=1 and endks=1 and KS1READPS_P>0 and KS1WRITPS_P>0 and KS1MATPS_P>0 then (((KS1READPS_P+KS1WRITPS_P)/2) + (KS1MATPS_P))/2
when schres=1 and endks=1 and KS1READPS_P>0 and KS1WRITPS_P>0 and KS1MATPS_P=0 or KS1MATPS_P is NULL then (KS1READPS_P+KS1WRITPS_P)/2
when schres=1 and endks=1 and KS1READPS_P=0 or KS1READPS_P is NULL and KS1WRITPS_P>0 and KS1MATPS_P>0 then (KS1WRITPS_P+KS1MATPS_P)/2
when schres=1 and endks=1 and KS1WRITPS_P=0 or KS1WRITPS_P is NULL and KS1READPS_P>0 and KS1MATPS_P>0 then (KS1READPS_P+KS1MATPS_P)/2
when schres=1 and endks=1 and KS1READPS_P>0 and KS1WRITPS_P=0 or KS1WRITPS_P is NULL and KS1MATPS_P=0 or KS1MATPS_P is NULL then KS1READPS_P
when schres=1 and endks=1 and KS1WRITPS_P>0 and KS1READPS_P=0 or KS1READPS_P is NULL and KS1MATPS_P=0 or KS1MATPS_P is NULL then KS1WRITPS_P
when schres=1 and endks=1 and KS1MATPS_P>0 and KS1WRITPS_P=0 or KS1WRITPS_P is NULL and KS1READPS_P=0 or KS1READPS_P is NULL then KS1MATPS_P
when schres=1 and endks=1 and KS1MATPS_P=0 or KS1MATPS_P is NULL and KS1WRITPS_P=0 or KS1WRITPS_P is NULL and KS1READPS_P=0 or KS1READPS_P is NULL then NULL
end as [KS1APS_pup]
into #ks1aps
from [dbo].[KS2_2016_Pupil_3rdamended_anonymised]
Where NFTYPE in(20,51,52,57,58)

Use AcademiesandSchoolOrganisation
go
Select table1.*, table2.[Chain ID], table2.[Chain Name], table2.[Chain Type]
into #ks1aps2
from #ks1aps as table1
left join [dbo].[Chains_29Sep2015_FULL_Mike] as table2
on table1.urn=table2.urn
Order by table1.urn ASC

Use AcademiesandSchoolOrganisation
go
Select [CHAIN ID], [CHAIN NAME], Sum(case when KS1APS_pup>0 then KS1APS_pup else NULL end) as KS1APS_TOT, Count(case when KS1APS_pup>0 then KS1APS_pup else NULL end) as KS1APS_CNT
into #ks1aps3
from #ks1aps2
group by [Chain ID], [CHAIN NAME]
Order by [Chain Name] ASC

Use AcademiesandSchoolOrganisation
go
Select [CHAIN ID], [CHAIN NAME], ([KS1APS_TOT]/[KS1APS_CNT]) as KS1APS_MAT
into #ks1aps4
from #ks1aps3
Order by [Chain Name] ASC

Use Assessment
go
Select urn, 
sum(case when schres=1 and endks=1 then 1 else 0 end) as TELIG,
sum(case when schres=1 and endks=1 and FSM6CLA1A=1 then 1 else 0 end) as TFSMCLA, --disadvantaged pupils (wider) so FSM6, looked after and adopted from care
sum(case when schres=1 and endks=1 and SENF in ('S','E', 'P') then 1 else 0 end) as SENELS, --this captures pupils with special educational need statemented or EHC or school action plus
sum(case when schres=1 and endks=1 and LANG1ST in ('OTB','OTH') then 1 else 0 end) as TEALGRP2 --this captures the number of pupils with english as an additional language
into #provks2context1
from [dbo].[KS2_2016_Pupil_3rdamended_anonymised]
Where NFTYPE in(20,51,52,57,58)
Group by urn
Order by urn ASC

Use AcademiesandSchoolOrganisation
Go
Select table1.*, table2.[Chain ID], table2.[Chain Name], table2.[Chain Type]
into #provks2context2
from #provks2context1 as table1
left join [dbo].[Chains_29Sep2015_FULL_Mike] as table2
on table1.urn=table2.urn
Order by table1.urn ASC

Use academiesandschoolorganisation
Go
Select [Chain ID], [Chain Name], Sum(TELIG) as [TELIG_MAT], Sum(TFSMCLA) as [TFSMCLA_MAT], Sum(SENELS) as [SENELS_MAT], Sum(TEALGRP2) as [TEALGRP2_MAT], count(case when [telig]>0 then (urn) else NULL end) as [No. MAT schools]
into #provks2context3
from #provks2context2
Where [Chain Name] is not null
Group by [Chain ID], [Chain Name]
Order by [Chain Name] ASC

Use academiesandschoolorganisation
Go
Select [Chain ID], [Chain Name], round((Cast([TFSMCLA_MAT]as float)/nullif(Cast([TELIG_MAT] as float),0))*100,1) as [% "disadvantaged"], round((Cast([SENELS_MAT] as float)/nullif(Cast([TELIG_MAT] as float),0))*100,1) as [% SEN/Statemented/EHC/School Action +], round((Cast([TEALGRP2_MAT] as float)/nullif(Cast([TELIG_MAT] as float),0))*100,1) as [% English as aditional language]
into #provks2context4
from #provks2context3

/****************************************************************************/

--READING LEVEL MAT SCORES FOR KEY STAGE 2
--This works out the school level reading progress score and number of pupils included and puts them into a temporary table

Use Assessment
Go
Select urn, round(avg(READPROGSCORE_P),1) as 'School level reading progress score', sum(INREADPROG) as 'Number of pupils included within reading progress', Sum(SCHRES) as 'Number of eligible pupils'
into #readprog1
from [dbo].[KS2_2016_Pupil_3rdamended_anonymised]
Where NFTYPE in(20,51,52,57,58)
Group by urn
Order by urn ASC

--This isolates all the school level reading scores that are not NULL and puts them into a temporary table

Select *
into #readprog2
from #readprog1
where [School level reading progress score] is not null
Order by urn ASC

--This joins on the chain infomation for each school within #readprog2

Use AcademiesandSchoolOrganisation
Go
Select table1.*, table2.[Chain ID], table2.[Chain Name], table2.[Chain Type], table2.[Date Joined], table2.[Academy Type]
into #readprog3
from #readprog2 as table1
left join [dbo].[Chains_29Sep2015_FULL_Mike] as table2
on table1.urn=table2.urn
order by urn ASC

--This works out how many academic years each school has been open and in MAT and caps the years in chain to 4 years. MANUAL CHANGE TO YEAR WEIGHTINGS

Use AcademiesandSchoolOrganisation
Go
Select *, case 
when [Date Joined]>'11-Sep-2015' then 0 
when [Date Joined]>='12-Sep-2014' and [Date Joined]<='11-Sep-2015' then 0
when [Date Joined]>='12-Sep-2013' and [Date Joined]<='11-Sep-2014' then 0
When [Date Joined]>='12-Sep-2012' and [Date Joined]<='11-Sep-2013' then 3
When [Date Joined]<='11-Sep-2012' then 4
end as [Capped years in chain]
into #readprog4
from #readprog3
order by urn ASC

Use academiesandschoolorganisation
go
select *, case
When [Capped years in chain]>=3 and [Academy Type] in ('Academy Converter') then 1 else 0
end as [Number academy converters],
Case
when [Capped years in chain]>=3 and [Academy type] in ('Academy Sponsor Led') then 1 else 0
end as [Number academy sponsor led],
Case
when [Capped years in chain]>=3 and [Academy type] in ('Free Schools','Studio Schools','University Technical College') then 1 else 0
end as [Number free schools]
into #readprog5
from #readprog4
order by urn ASC

--This calculates the reading score weights at school level and isolates only those that have been in a chain for 1 year or more

Use AcademiesandSchoolOrganisation
Go
Select *, [Number of eligible pupils]*[Capped years in chain] as [Reading weight], ([Number of eligible pupils]*[Capped years in chain])*[School level reading progress score] as [Reading weighted score]
into #readprog6
from #readprog5
Where [Capped years in chain]>=1
order by urn ASC

--This calculates the MAT level numerators and denominators for the reading measure regardless of MAT size

Use AcademiesandSchoolOrganisation
Go
Select [Chain ID], [Chain Name], Count([urn]) as [Schools included], sum([Number academy converters]) as [Converter academies], sum([Number academy sponsor led]) as [Sponsored academies], sum([Number free schools]) as [Free Schools, UTCs & Studio Schools],
sum(case [Capped years in chain] when '1' then 1 else 0 end) as [1 year], sum(case [Capped years in chain] when '2' then 1 else 0 end) as [2 years], sum(case [Capped years in chain] when '3' then 1 else 0 end) as [3 years], sum(case [Capped years in chain] when '4' then 1 else 0 end) as [4 or more years],
sum([Reading weight]) as [MAT reading weight], sum([Reading weighted score]) as [MAT reading weighted score], sum([Number of pupils included within reading progress]) as [Number of pupils included in reading MAT measure]
into #readprog7
from #readprog6
group by [Chain ID], [Chain Name]

--This filters only those MATs greater than or equal to 3 in size and calculates their overall reading score

Use AcademiesandSchoolOrganisation
Go
Select [Chain ID], [Chain Name], [Schools included], [Converter academies], [Sponsored academies], [Free Schools, UTCs & Studio Schools], 
[1 year], [2 years], [3 years], [4 or more years],
[Number of pupils included in reading MAT measure], [MAT reading weighted score]/[MAT reading weight] as [Reading progress measure]
into #readprog8
from #readprog7
Where [Schools included]>=3

--This adds on the reading variance across all pupils nationally

Use AcademiesandSchoolOrganisation
Go
Select table1.*, table2.[Reading progress score variance]
into #readprog9
from #readprog8 as table1
left join #nationalvariances as table2
on 1=1
Where [Schools included]>=3

--This adds on the confidence interval, lower and upper limits

Use AcademiesandSchoolOrganisation
Go
Select *, 1.96*sqrt([Reading progress score variance]/[Number of pupils included in reading MAT measure]) as [Confidence interval width],
[Reading progress measure]-(1.96*sqrt([Reading progress score variance]/[Number of pupils included in reading MAT measure])) as [lower confidence limit],
[Reading progress measure]+(1.96*sqrt([Reading progress score variance]/[Number of pupils included in reading MAT measure])) as [upper confidence limit]
into #readprog10
from #readprog9
Where [Schools included]>=3

--This pulls out the bear minimum final reading measure data we need

Use AcademiesandSchoolOrganisation
Go
Select [Chain ID], [Chain Name], [Schools included], [Converter academies], [Sponsored academies], [Free Schools, UTCs & Studio Schools], 
[1 year], [2 years], [3 years], [4 or more years],
[Reading progress measure], [Confidence interval width], [lower confidence limit], [upper confidence limit]
into #readprogmea
from #readprog10
Where [Schools included]>=3

/****************************************************************************/

/****************************************************************************/

--WRITING LEVEL MAT SCORES FOR KEY STAGE 2
--This works out the school level writing progress score and number of pupils included and puts them into a temporary table

Use Assessment
Go
Select urn, round(avg(WRITPROGSCORE_P),1) as 'School level writing progress score', sum(INWRITPROG) as 'Number of pupils included within writing progress', Sum(SCHRES) as 'Number of eligible pupils'
into #writprog1
from [dbo].[KS2_2016_Pupil_3rdamended_anonymised]
Where NFTYPE in(20,51,52,57,58)
Group by urn
Order by urn ASC

--This isolates all the school level writing scores that are not NULL and puts them into a temporary table

Select *
into #writprog2
from #writprog1
where [School level writing progress score] is not null
Order by urn ASC

--This joins on the chain infomation for each school within #writprog2

Use AcademiesandSchoolOrganisation
Go
Select table1.*, table2.[Chain ID], table2.[Chain Name], table2.[Chain Type], table2.[Date Joined], table2.[Academy Type]
into #writprog3
from #writprog2 as table1
left join [dbo].[Chains_29Sep2015_FULL_Mike] as table2
on table1.urn=table2.urn
order by urn ASC

--This works out how many academic years each school has been open and in MAT and caps the years in chain to 4 years. MANUAL CHANGE TO YEAR WEIGHTINGS

Use AcademiesandSchoolOrganisation
Go
Select *, case 
when [Date Joined]>'11-Sep-2015' then 0 
when [Date Joined]>='12-Sep-2014' and [Date Joined]<='11-Sep-2015' then 0
when [Date Joined]>='12-Sep-2013' and [Date Joined]<='11-Sep-2014' then 0
When [Date Joined]>='12-Sep-2012' and [Date Joined]<='11-Sep-2013' then 3
When [Date Joined]<='11-Sep-2012' then 4
end as [Capped years in chain]
into #writprog4
from #writprog3
order by urn ASC

Use academiesandschoolorganisation
go
select *, case
When [Capped years in chain]>=3 and [Academy Type] in ('Academy Converter') then 1 else 0
end as [Number academy converters],
Case
when [Capped years in chain]>=3 and [Academy type] in ('Academy Sponsor Led') then 1 else 0
end as [Number academy sponsor led],
Case
when [Capped years in chain]>=3 and [Academy type] in ('Free Schools','Studio Schools','University Technical College') then 1 else 0
end as [Number free schools]
into #writprog5
from #writprog4
order by urn ASC

--This calculates the writing score weights at school level and isolates only those that have been in a chain for 1 year or more

Use AcademiesandSchoolOrganisation
Go
Select *, [Number of eligible pupils]*[Capped years in chain] as [Writing weight], ([Number of eligible pupils]*[Capped years in chain])*[School level writing progress score] as [Writing weighted score]
into #writprog6
from #writprog5
Where [Capped years in chain]>=1
order by urn ASC

--This calculates the MAT level numerators and denominators for the writing measure regardless of MAT size

Use AcademiesandSchoolOrganisation
Go
Select [Chain ID], [Chain Name], Count([urn]) as [Schools included], sum([Number academy converters]) as [Converter academies], sum([Number academy sponsor led]) as [Sponsored academies], sum([Number free schools]) as [Free Schools, UTCs & Studio Schools],
sum(case [Capped years in chain] when '1' then 1 else 0 end) as [1 year], sum(case [Capped years in chain] when '2' then 1 else 0 end) as [2 years], sum(case [Capped years in chain] when '3' then 1 else 0 end) as [3 years], sum(case [Capped years in chain] when '4' then 1 else 0 end) as [4 or more years],
sum([Writing weight]) as [MAT Writing weight], sum([Writing weighted score]) as [MAT writing weighted score], sum([Number of pupils included within writing progress]) as [Number of pupils included in writing MAT measure]
into #writprog7
from #writprog6
group by [Chain ID], [Chain Name]

--This filters only those MATs greater than or equal to 3 in size and calculates their overall writing score

Use AcademiesandSchoolOrganisation
Go
Select [Chain ID], [Chain Name], [Schools included], [Converter academies], [Sponsored academies], [Free Schools, UTCs & Studio Schools], 
[1 year], [2 years], [3 years], [4 or more years],
[Number of pupils included in writing MAT measure], [MAT writing weighted score]/[MAT writing weight] as [Writing progress measure]
into #writprog8
from #writprog7
Where [Schools included]>=3

--This adds on the writing variance across all pupils nationally

Use AcademiesandSchoolOrganisation
Go
Select table1.*, table2.[Writing progress score variance]
into #writprog9
from #writprog8 as table1
left join #nationalvariances as table2
on 1=1
Where [Schools included]>=3

--This adds on the confidence interval, lower and upper limits

Use AcademiesandSchoolOrganisation
Go
Select *, 1.96*sqrt([Writing progress score variance]/[Number of pupils included in writing MAT measure]) as [Confidence interval width],
[Writing progress measure]-(1.96*sqrt([Writing progress score variance]/[Number of pupils included in writing MAT measure])) as [lower confidence limit],
[Writing progress measure]+(1.96*sqrt([Writing progress score variance]/[Number of pupils included in writing MAT measure])) as [upper confidence limit]
into #writprog10
from #writprog9
Where [Schools included]>=3

--This pulls out the bear minimum final writing measure data we need

Use AcademiesandSchoolOrganisation
Go
Select [Chain ID], [Chain Name], [Schools included], [Converter academies], [Sponsored academies], [Free Schools, UTCs & Studio Schools], 
[1 year], [2 years], [3 years], [4 or more years],
[Writing progress measure], [Confidence interval width], [lower confidence limit], [upper confidence limit]
into #writprogmea
from #writprog10
Where [Schools included]>=3

/****************************************************************************/

/****************************************************************************/

--MATHS LEVEL MAT SCORES FOR KEY STAGE 2
--This works out the school level maths progress score and number of pupils included and puts them into a temporary table

Use Assessment
Go
Select urn, round(avg(MATPROGSCORE_P),1) as 'School level maths progress score', sum(INMATPROG) as 'Number of pupils included within maths progress', Sum(SCHRES) as 'Number of eligible pupils'
into #mathprog1
from [dbo].[KS2_2016_Pupil_3rdamended_anonymised]
Where NFTYPE in(20,51,52,57,58)
Group by urn
Order by urn ASC

--This isolates all the school level maths scores that are not NULL and puts them into a temporary table

Select *
into #mathprog2
from #mathprog1
where [School level maths progress score] is not null
Order by urn ASC

--This joins on the chain infomation for each school within #mathprog2

Use AcademiesandSchoolOrganisation
Go
Select table1.*, table2.[Chain ID], table2.[Chain Name], table2.[Chain Type], table2.[Date Joined], table2.[Academy Type]
into #mathprog3
from #mathprog2 as table1
left join [dbo].[Chains_29Sep2015_FULL_Mike] as table2
on table1.urn=table2.urn
order by urn ASC

--This works out how many academic years each school has been open and in MAT and caps the years in chain to 4 years. MANUAL CHANGE TO YEAR WEIGHTINGS

Use AcademiesandSchoolOrganisation
Go
Select *, case 
when [Date Joined]>'11-Sep-2015' then 0 
when [Date Joined]>='12-Sep-2014' and [Date Joined]<='11-Sep-2015' then 0
when [Date Joined]>='12-Sep-2013' and [Date Joined]<='11-Sep-2014' then 0
When [Date Joined]>='12-Sep-2012' and [Date Joined]<='11-Sep-2013' then 3
When [Date Joined]<='11-Sep-2012' then 4
end as [Capped years in chain]
into #mathprog4
from #mathprog3
order by urn ASC

Use academiesandschoolorganisation
go
select *, case
When [Capped years in chain]>=3 and [Academy Type] in ('Academy Converter') then 1 else 0
end as [Number academy converters],
Case
when [Capped years in chain]>=3 and [Academy type] in ('Academy Sponsor Led') then 1 else 0
end as [Number academy sponsor led],
Case
when [Capped years in chain]>=3 and [Academy type] in ('Free Schools','Studio Schools','University Technical College') then 1 else 0
end as [Number free schools]
into #mathprog5
from #mathprog4
order by urn ASC

--This calculates the math score weights at school level and isolates only those that have been in a chain for 1 year or more

Use AcademiesandSchoolOrganisation
Go
Select *, [Number of eligible pupils]*[Capped years in chain] as [Maths weight], ([Number of eligible pupils]*[Capped years in chain])*[School level maths progress score] as [Maths weighted score]
into #mathprog6
from #mathprog5
Where [Capped years in chain]>=1
order by urn ASC

--This calculates the MAT level numerators and denominators for the maths measure regardless of MAT size

Use AcademiesandSchoolOrganisation
Go
Select [Chain ID], [Chain Name], Count([urn]) as [Schools included], sum([Number academy converters]) as [Converter academies], sum([Number academy sponsor led]) as [Sponsored academies], sum([Number free schools]) as [Free Schools, UTCs & Studio Schools],
sum(case [Capped years in chain] when '1' then 1 else 0 end) as [1 year], sum(case [Capped years in chain] when '2' then 1 else 0 end) as [2 years], sum(case [Capped years in chain] when '3' then 1 else 0 end) as [3 years], sum(case [Capped years in chain] when '4' then 1 else 0 end) as [4 or more years],
sum([Maths weight]) as [MAT maths weight], sum([Maths weighted score]) as [MAT maths weighted score], sum([Number of pupils included within maths progress]) as [Number of pupils included in maths MAT measure]
into #mathprog7
from #mathprog6
group by [Chain ID], [Chain Name]

--This filters only those MATs greater than or equal to 3 in size and calculates their overall maths score

Use AcademiesandSchoolOrganisation
Go
Select [Chain ID], [Chain Name], [Schools included], [Converter academies], [Sponsored academies], [Free Schools, UTCs & Studio Schools], 
[1 year], [2 years], [3 years], [4 or more years],
[Number of pupils included in maths MAT measure], [MAT maths weighted score]/[MAT maths weight] as [Maths progress measure]
into #mathprog8
from #mathprog7
Where [Schools included]>=3

--This adds on the maths variance across all pupils nationally

Use AcademiesandSchoolOrganisation
Go
Select table1.*, table2.[Maths progress score variance]
into #mathprog9
from #mathprog8 as table1
left join #nationalvariances as table2
on 1=1
Where [Schools included]>=3

--This adds on the confidence interval, lower and upper limits

Use AcademiesandSchoolOrganisation
Go
Select *, 1.96*sqrt([Maths progress score variance]/[Number of pupils included in maths MAT measure]) as [Confidence interval width],
[Maths progress measure]-(1.96*sqrt([Maths progress score variance]/[Number of pupils included in maths MAT measure])) as [lower confidence limit],
[Maths progress measure]+(1.96*sqrt([Maths progress score variance]/[Number of pupils included in maths MAT measure])) as [upper confidence limit]
into #mathprog10
from #mathprog9
Where [Schools included]>=3

--This pulls out the bear minimum final maths measure data we need

Use AcademiesandSchoolOrganisation
Go
Select [Chain ID], [Chain Name], [Schools included], [Converter academies], [Sponsored academies], [Free Schools, UTCs & Studio Schools], 
[1 year], [2 years], [3 years], [4 or more years],
[Maths progress measure], [Confidence interval width], [lower confidence limit], [upper confidence limit]
into #mathprogmea
from #mathprog10
Where [Schools included]>=3

/****************************************************************************/
/****************************************************************************/

--This section of code pulls together our final output

Use AcademiesandSchoolOrganisation
Go
Select table1.[Chain ID], table1.[Chain Name], 
table7.[TELIG_MAT], table7.[No. MAT Schools], table8.[KS1APS_MAT], table4.[% "disadvantaged"], table4.[% SEN/Statemented/EHC/School Action +], table4.[% English as aditional language],
table1.[Schools included] as 'Schools included in reading measure', table1.[Converter academies], table1.[Sponsored academies], table1.[Free Schools, UTCs & Studio Schools],
table1.[1 year], table1.[2 years], table1.[3 years], table1.[4 or more years],
round(table1.[Reading progress measure],1) as [Reading progress measure], table1.[Confidence interval width], table1.[lower confidence limit], table1.[upper confidence limit], 
case when round(table1.[lower confidence limit],10)>0 then 'Significantly above average' when round(table1.[upper confidence limit],10)<0 then 'Significantly below average' when table1.[lower confidence limit] is NULL then NULL when table1.[upper confidence limit] is NULL then NULL else 'Close to average' end as [Measure description],
table2.[Schools included] as 'Schools included in writing measure', table2.[Converter academies], table2.[Sponsored academies], table2.[Free Schools, UTCs & Studio Schools],
table2.[1 year], table2.[2 years], table2.[3 years], table2.[4 or more years],
round(table2.[Writing progress measure],1) as [Writing progress measure], table2.[Confidence interval width], table2.[lower confidence limit], table2.[upper confidence limit],
case when round(table2.[lower confidence limit],10)>0 then 'Significantly above average' when round(table2.[upper confidence limit],10)<0 then 'Significantly below average' when table2.[lower confidence limit] is NULL then NULL when table2.[upper confidence limit] is NULL then NULL else 'Close to average' end as [Measure description],
table3.[Schools included] as 'Schools included in maths measure', table3.[Converter academies], table3.[Sponsored academies], table3.[Free Schools, UTCs & Studio Schools],
table3.[1 year], table3.[2 years], table3.[3 years], table3.[4 or more years],
round(table3.[Maths progress measure],1) as [Maths progress measure], table3.[Confidence interval width], table3.[lower confidence limit], table3.[upper confidence limit],
case when round(table3.[lower confidence limit],10)>0 then 'Significantly above average' when round(table3.[upper confidence limit],10)<0 then 'Significantly below average' when table3.[lower confidence limit] is NULL then NULL when table3.[upper confidence limit] is NULL then NULL else 'Close to average' end as [Measure description]
From #readprogmea as table1
Full join #writprogmea as table2
on table1.[Chain ID]=table2.[Chain ID]
full join #mathprogmea as table3
on table1.[Chain ID]=table3.[Chain ID]
left join #provks2context4 as table4
on table1.[Chain ID]=table4.[Chain ID]
left join #provks2context3 as table7
on table1.[Chain ID]=table7.[Chain ID]
left join #ks1aps4 as table8
on table1.[Chain ID]=table8.[Chain ID]
Order by [Chain Name] ASC
