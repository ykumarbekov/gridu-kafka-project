### Kafka Capstone project  
Part 1: Preparing data   

Part 2: Aggregation and saving data  
Options: KafkaStreams; KSQL/KSQLDB  

Part 3: KafkaStreams application: pushes in a topic for each day the hour-slot with the highest amount of commits  
___
(KSQL/KSQLDB)  

Input message structure, example:  
```
{
"repo": "homebrew",
"commit_id": "39ee15224f0a6702a9c3fd3c98f2ddea9cb141a6",
"commit": {"name": "Tom Preston-Werner", "email": "tom@mojombo.com", "date": "2011-06-28T01:41:27Z"},
"committer": {"name": "Tom Preston-Werner", "email": "tom@mojombo.com", "date": "2011-06-28T01:41:27Z"},
"owner": {"login": "mojombo", "name": "Tom Preston-Werner"},
"contributor": 0,
"language": "Ruby"
}
```
Use docker-compose file to build infrastructure and run cli  
```
docker-compose -f zk-broker-ksqldb.yml up -d
docker exec -it ksqldb-cli ksql http://ksqldb-server:8088

create or replace stream COMMITS_DATA_1STR (
   repo VARCHAR,
   commit_id VARCHAR,
   commit STRUCT<name VARCHAR, email VARCHAR, date VARCHAR>,
   committer STRUCT<name VARCHAR, email VARCHAR, date VARCHAR>,
   owner STRUCT<login VARCHAR, name VARCHAR>,
   contributor INT,
   language VARCHAR
   ) with (kafka_topic='COMMITS-DATA', value_format='json', partitions=1);

create or replace stream COMMITS_DATA_2STR as
    select
    repo,
    commit_id,
    commit->name as commit_person_name,
    commit->email as commit_person_email,
    commit->date as commit_date,
    committer->name as committer_name,
    committer->email as committer_email,
    committer->date as committer_date,
    owner->login as owner_login,
    contributor,
    language
from COMMITS_DATA_1STR;
```
- Top 5 contributors by number of commits  
```
  create or replace table COMMITS_CONTRIB as
  select  
  COMMIT_PERSON_NAME,
  count(1) as COUNT
  from COMMITS_DATA_2STR 
  where CONTRIBUTOR=1
  group by COMMIT_PERSON_NAME EMIT CHANGES;

  create or replace stream COMMITS_CONTRIB_STR (COMMIT_PERSON_NAME VARCHAR KEY, COUNT BIGINT) with (kafka_topic='COMMITS_CONTRIB', value_format='json')
      
  create table TOP5_COMMITS_CONTRIB as
  select
  COMMIT_PERSON_NAME,
  topk(COUNT, 5)
  from COMMITS_CONTRIB_STR
  group by COMMIT_PERSON_NAME;
```
- Total number of commits  
*,and we can use create table ... as select ... and receive daily total commits number*
```
create or replace stream COMMITS_DATA_3STR as 
  select timestamptostring(rowtime, 'yyyy-MM-dd') as dt, commit_id from COMMITS_DATA_2STR;

select dt, count(commit_id) as TOTAL from COMMITS_DATA_3STR group by dt emit changes;
```
- Total number of committers
```
create or replace stream COMMITS_DATA_4STR as 
  select timestamptostring(rowtime, 'yyyy-MM-dd') as dt, committer_name from COMMITS_DATA_2STR;

  select dt, count_distinct(committer_name) as TOTAL_COMMITTER from COMMITS_DATA_4STR group by dt emit changes;
```
- Total number of commits for each programming language
```
select language, count(commit_id) as TOTAL from COMMITS_DATA_2STR group by language emit changes;
```






