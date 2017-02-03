#HSLIDE

### SQL injection in many parameters

#HSLIDE

### Vulnerable page

- web app with filtering capabilities
- basic test `' or '1'='1` returned more results
- burp potential finding listed
- sqlmap failed to confirm the vulnerability, due to constraints

#HSLIDE

### Constraints

- param0 - 50 characters
- param1 - 50 characters
- param2 - 20 characters

and so on...

#HSLIDE

### Guessing phase

```
SELECT <STH> from <STH> where param1 = '<PARAM1>' 
    and param2 = '<PARAM2>'
    and param3 = '<PARAM3>' and ......
```

#HSLIDE

### Additional conditions?

- type of query
- multiline/single line query
- no casting in parameters, pure string params
- MySQL/MSSQL/PostgreSQL

#HSLIDE

### Conclusions and assumptions

- it's a filter so it has to be `SELECT`
- assumption was multiline, but later turned out to be single-line
- some of the fields were numbers, but fortunately no casting
- DB was known, but just to verify `@@VERSION` used to get the banner
    - manually ;-(
    - sqlmap could not automate it, even said there is no SQLi

#HSLIDE

### How to make an automated attack?

#HSLIDE

### How to get more characters?

#HSLIDE

### First idea?

#HSLIDE

### Comments!

#HSLIDE

### Single-line comment and payload

```sql
-- SELECT * from table where name like '%<PAYLOAD>%'
-- PAYLOAD = %'--
SELECT * from table where name like '%%'--%'
```

Good if each parameter would be in separate line, but then new lines, empty spaces and so on.

#HSLIDE

### Next comments style

#HSLIDE

### Multiline/fragmented

```sql
/* multi
   iline */

//Fragmented
SELECT * from table where 
    /* 
       here is the first condition
    */ name like '...' and /* Here 
    is the second param */
    and gender = '...'
```

#HSLIDE

### It can be useful, but how to glue it together?

- concatenate many parameters
- figure out the order

#HSLIDE

### Ordering tip

Used single-line comment to figure out the order

```sql
-- Two payloads
--   Good payload '-- to comment out the entire line
--   Bad payload  ' to break the query
-- sample query
SELECT * from table where param0 = '<PAYLOAD0>' and param1 = '<PAYLOAD1>'
```

### Tests and conclusions

- PAYLOAD0=Bad & PAYLOAD1=Good & query-status='SQL error, blind error, syntax error'
    - PAYLOAD0 broke the query
- PAYLOAD0=Good & PAYLOAD1=Bad & query-status='SQL ok'
    - PAYLOAD0 commented out the line and prevented syntax error

#HSLIDE


### So the final payload? (1)

```sql
SELECT * from table 
     where param0='<INJECTION_PART0><SUFFIX0>' 
     and param1 = '<PREFIX1><INJECTION_PART1><SUFFIX1>'

-- SUFFIX0 = '/*
-- PREFIX1 = */
-- SUFFIX1 = '--
```

### So the final payload? (2)

```sql
SELECT * from table
     where param0='XXXXXXXXXX_50_CHARS_MINUS_SUFFIXXXX /*' 
     and param1 = '/* YYYYY_50_CHARS_MINUS_SUFF_AND_PREFIX'--'

-- query transformed to
SELECT * from table where param0 = '50_CHARS-suff0_PLUS_50CHARS_MINUS_suff1_PLUS_pref1'
```

#HSLIDE

### Automation!

- sqlmap can't do it... alone
- tamper scripts

#HSLIDE

### Grand idea

- split sqlmap's payload - white chars
- concatenate parts until **len<50-len(suffix0)**, that is `param0`
- concatenate parts until **len<50-len(suffix1)-len(prefix1)**
- and so on for more parameters

