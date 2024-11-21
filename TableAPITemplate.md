## How to Use This Template
- **Invocation:** Start by saying, "Please use my Table API template."
- **Step 1:** I will ask for the table definitions.
- **Step 2:** You provide the required information as attachments or directly in the chat.
- **Step 3:** I generate and return the SQL functions based on the template.
- **Step 4:** Review the output and request any adjustments if needed.

# Table API Template

## Purpose
This template guides the creation of PostgreSQL functions for new tables in our database. Each table requires two functions:
1. **List Function:** Retrieves a filtered list of rows.
2. **Single Row Function:** Retrieves a single row, often with joined details.

---

## Rules and Standards
### **1. Absolute MUST, Non-Negotiable**

- All functions **MUST**: 
	1. Use `postgres` as the owner in the `ALTER FUNCTION` statement. 
	2. Include explicit `NULL::<type>` for every column in the `IF NOT FOUND` block. 
	3. Follow exact column orders in `RETURNS TABLE`.
### **2. `DROP` Block**
Every function must include a `DROP FUNCTION` block at the top to ensure older versions are replaced. For example:
```sql
DROP FUNCTION IF EXISTS <schema>.<function_name>(<param_types>);
```

### **3. `Permissions` Block**
Each function must include a permissions section to set ownership and grant execution rights. For example:
```sql
-- Set Ownership
ALTER FUNCTION <schema>.<function_name>(<param_types>)
OWNER TO postgres;

-- Grant Execution Permissions
GRANT EXECUTE ON FUNCTION <schema>.<function_name>(<param_types>)
TO postgres;
```

### **4. Ownership

```
In every case, the permissions and ownership sections should assign to a user named postgres. 

And so, references to <owner> should translate to postgres.
```


### **5. Column Names

Column Names should always be verbose. For example, this is not acceptable...
```sql
COALESCE(
	(
		SELECT 
		json_agg(cd) 
		FROM public.crew_dtl cd 
		WHERE cd.crew_hdr_id = ch.crew_hdr_id 
	), '[]'::json
) AS crew_details
```

It should be written with explicit names. The reason for this is that you will be creating these functions in the most generic, malleable way possible.  It's rare that I will actually want ALL of the columns. But having you write it as such will make it very easy for me to remove the columns I do not need.
```sql
COALESCE(
	(
		SELECT 
		json_agg(cd) 
		FROM (
			SELECT
			ROW_NUMBER() OVER (ORDER BY cd.crew_dtl_id) AS marker_number,
			cd.crew_dtl_id,
			cd.entity_id,
			cd.created_by,
			CONCAT(cr.last_name, ', ', cr.first_name) AS created_by_name,
			cd.created_date,
			cd.updated_by,
	        CONCAT(up.last_name, ', ', up.first_name) AS updated_by_name,
	        cd.updated_date
	        FROM public.crew_dtl cd
	        LEFT OUTER JOIN public.app_user cr
			ON cr.user_id = cd.created_by
			LEFT OUTER JOIN public.app_user up
			ON up.user_id = cd.updated_by
			WHERE cd.crew_hdr_id = ch.crew_hdr_id
		) 
		WHERE cd.crew_hdr_id = ch.crew_hdr_id 
	), '[]'::json
) AS crew_details
```

### **6. Not Found Block

This is not acceptable:
```sql
IF NOT FOUND THEN 
	RETURN QUERY 
	SELECT 
		NULL, NULL, NULL, NULL, 		 
		'[]'::json; 
END IF;
```

Rather, we need explicit data types to match whatever the designated structure is for the return. And, each field should have the name of the field at the end of the line as a comment, as shown here:
```sql
IF NOT FOUND THEN 
	RETURN QUERY 
	SELECT 
		FALSE::boolean, -- function_status
		'<Fail Message>'::text, -- msg
		NULL::bigint, -- crew_dtl_id 
		NULL::bigint, -- entity_id
		'[]'::json; 
END IF;

```

**IF NOT FOUND Formatting:**
  - Each `NULL` field in the `IF NOT FOUND` block must be on a new line, with a comment indicating its corresponding column.
  - This improves readability and ensures alignment with the `RETURNS TABLE` clause.

Example:
```sql
IF NOT FOUND THEN
    RETURN QUERY SELECT 
        NULL::BIGINT,            -- customer_id
        NULL::BIGINT,            -- org_id
        NULL::SMALLINT,          -- customer_type_id
        NULL::CHARACTER VARYING, -- customer_name
        NULL::BOOLEAN,           -- tax_exempt
        NULL::BIGINT,            -- address_id
        NULL::SMALLINT,          -- customer_status_id
        NULL::TEXT,              -- created_by_name
        NULL::TEXT,              -- updated_by_name
        '[]'::JSON;              -- notes
END IF;
```

### **7. Initial Fields

Every query should have 2 fields at the beginning called function_status and msg.  This is not true for embedded detail table queries that are retrieved as json, just the main query.  These 2 required fields look like this when defined...
```sql
	function_status BOOLEAN,
	msg TEXT,
```

and look like this in the query...
```sql
	TRUE::boolean,
	'Customer found'::text,
```


---

## Template Functions

### **Template: List Function**
```sql

-- Replace <table_name> with the actual table name
-- Replace <header_key_field> with the primary key of the header table
-- <schema> should always be ilandscape
-- Replace <function_name> with the desired name of the function
-- Replace <param_name> placeholders with the appropriate parameter names
-- Replace <param_type> placeholders with the appropriate parameter types

DROP FUNCTION IF EXISTS <schema>.<function_name>(<param1_type>, <param2_type>);

CREATE OR REPLACE FUNCTION <schema>.<function_name>(
    <param1_name> <param1_type>, 
    <param2_name> <param2_type>
)
RETURNS TABLE(
    function_status BOOLEAN, 
    msg TEXT, 
    <columns...>,
    created_by_name TEXT,
    updated_by_name TEXT
)
LANGUAGE plpgsql
AS $function$
BEGIN
    RETURN QUERY
    SELECT
        TRUE::BOOLEAN,
        'Success message here'::TEXT,
        <column_names...>,
        CONCAT(cr.last_name, ', ', cr.first_name) AS created_by_name,
        CONCAT(up.last_name, ', ', up.first_name) AS updated_by_name
    FROM public.<table_name> t
    LEFT OUTER JOIN public.app_user cr
    ON cr.user_id = t.created_by
    LEFT OUTER JOIN public.app_user up
    ON up.user_id = t.updated_by
    WHERE t.<filter_column> = <filter_value>;

    IF NOT FOUND THEN
        RETURN QUERY 
        SELECT 
            FALSE::BOOLEAN, -- function_status
            'No data found for <table_name>' || <param2_name>::TEXT, -- msg
            <NULL replacements for columns>,
            NULL::text, -- created_by_name
            NULL::text; -- updated_by_name
    END IF;
END;
$function$;

-- Set Ownership
-- WARNING: Owner MUST always be `postgres`.
ALTER FUNCTION <schema>.<function_name>(<param1_type>, <param2_type>) OWNER TO postgres;

-- Grant Execution Permissions
-- WARNING: MUST always grant execute to `postgres`.
GRANT EXECUTE ON FUNCTION <schema>.<function_name>(<param1_type>, <param2_type>) TO postgres;
```

### **Template: Single Row Function**
```sql
DROP FUNCTION IF EXISTS <schema>.<function_name>(<param1_type>, <param2_type>, <param3_type>);

CREATE OR REPLACE FUNCTION <schema>.<function_name>(
    <param1_name> <param1_type>, 
    <param2_name> <param2_type>,
    <param3_name> <param3_type>
)
RETURNS TABLE(
    function_status BOOLEAN, 
    msg TEXT, 
    <header_table_columns...>,
    created_by_name TEXT,
    updated_by_name TEXT,
    <detail_table_name> JSON
)
LANGUAGE plpgsql
AS $function$
BEGIN
    RETURN QUERY
    SELECT
        TRUE::BOOLEAN,
        'Success message here'::TEXT,
        <column_names...>,
        CONCAT(cr.last_name, ', ', cr.first_name) AS created_by_name,
        CONCAT(up.last_name, ', ', up.first_name) AS updated_by_name,
        COALESCE(
            (SELECT json_agg(<detail_table_name_rows>) 
             FROM (
                 SELECT 
                     ROW_NUMBER() OVER (ORDER BY <sort_field>) AS marker_number,
                     <dtl_columns> 
                 FROM <detail_table>
                 WHERE <header_key_field> = <detail_foreign_key>
                 ORDER BY <sort_field>
             ) AS <detail_table_name>_rows),
            '[]'::json
        ) AS <detail_table_name>
    FROM public.<table_name> t
    LEFT OUTER JOIN public.app_user cr
    ON cr.user_id = t.created_by
    LEFT OUTER JOIN public.app_user up
    ON up.user_id = t.updated_by
    WHERE t.<filter_column> = <filter_value>;

    IF NOT FOUND THEN
        RETURN QUERY 
        SELECT 
            FALSE::BOOLEAN, -- function_status
            'No data found for <table_name>' || <param2_name>::TEXT, -- msg
            <NULL replacements for columns>,
            NULL::text, -- created_by_name
            NULL::text, -- updated_by_name,
            '[]'::json -- <detail_table_name>
    END IF;
END;
$function$;

-- Set Ownership
-- WARNING: Owner MUST always be `postgres`.
ALTER FUNCTION <schema>.<function_name>(<param1_type>, <param2_type>, <param3_type>) OWNER TO postgres;

-- Grant Execution Permissions
-- WARNING: MUST always grant execute to `postgres`.
GRANT EXECUTE ON FUNCTION <schema>.<function_name>(<param1_type>, <param2_type>, <param3_type>) TO postgres;
```

---

## Sample Functions

### **Sample: List Function**
```sql
DROP FUNCTION IF EXISTS public.get_crew_hdr_list(bigint);

CREATE OR REPLACE FUNCTION public.get_crew_hdr_list(org_id BIGINT)
RETURNS TABLE(
	function_status BOOLEAN,
	msg TEXT,
    crew_hdr_id BIGINT,
    org_id BIGINT,
    crew_type_id SMALLINT,
    crew_name CHARACTER VARYING,
    driver_id BIGINT,
    head_count SMALLINT,
    hours_per_day_0 NUMERIC(5, 2),
    hours_per_day_1 NUMERIC(5, 2),
    hours_per_day_2 NUMERIC(5, 2),
    hours_per_day_3 NUMERIC(5, 2),
    hours_per_day_4 NUMERIC(5, 2),
    hours_per_day_5 NUMERIC(5, 2),
    hours_per_day_6 NUMERIC(5, 2),
    created_by_name TEXT,
    updated_by_name TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
	    TRUE::BOOLEAN,
	    'Crews Found'::TEXT,
        ch.crew_hdr_id,
        ch.org_id,
        ch.crew_type_id,
        ch.crew_name,
        ch.driver_id,
        ch.head_count,
        ch.hours_per_day_0,
        ch.hours_per_day_1,
        ch.hours_per_day_2,
        ch.hours_per_day_3,
        ch.hours_per_day_4,
        ch.hours_per_day_5,
        ch.hours_per_day_6,
        CONCAT(cr.last_name, ', ', cr.first_name) AS created_by_name,
        CONCAT(up.last_name, ', ', up.first_name) AS updated_by_name
    FROM public.crew_hdr ch
    LEFT JOIN public.app_user cr ON cr.user_id = ch.created_by
    LEFT JOIN public.app_user up ON up.user_id = ch.updated_by
    WHERE ch.org_id = org_id;

    IF NOT FOUND THEN
        RETURN QUERY 
        SELECT 
	        FALSE::BOOLEAN, -- function_status
            'No data found for Crew' || <param2_name>::TEXT, -- msg
	        NULL::BIGINT, -- crew_hdr_id
	        NULL::BIGINT, -- org_id
	        NULL::SMALLINT, -- crew_type_id
	        NULL::CHARACTER VARYING, -- crew_name
	        NULL::BIGINT, -- driver_id
	        NULL::SMALLINT, -- head_count
	        NULL::NUMERIC(5, 2), -- hours_per_day_0
	        NULL::NUMERIC(5, 2), -- hours_per_day_1
	        NULL::NUMERIC(5, 2), -- hours_per_day_2
	        NULL::NUMERIC(5, 2), -- hours_per_day_3
	        NULL::NUMERIC(5, 2), -- hours_per_day_4
	        NULL::NUMERIC(5, 2), -- hours_per_day_5
	        NULL::NUMERIC(5, 2), -- hours_per_day_6
	        NULL::TEXT, -- created_by_name
	        NULL::TEXT -- updated_by_name
    END IF;
END;
$$ LANGUAGE plpgsql;

ALTER FUNCTION public.get_crew_hdr_list(bigint) OWNER TO postgres;
GRANT EXECUTE ON FUNCTION public.get_crew_hdr_list(bigint) TO postgres;
```

### **Sample: Single Row Function**
```sql
DROP FUNCTION IF EXISTS public.get_crew_hdr_details(bigint);

CREATE OR REPLACE FUNCTION public.get_crew_hdr_details(crew_hdr_id BIGINT)
RETURNS TABLE(
	function_status::BOOLEAN,
	msg::TEXT,
    crew_hdr_id BIGINT,
    org_id BIGINT,
    crew_type_id SMALLINT,
    crew_name CHARACTER VARYING,
    driver_id BIGINT,
    head_count SMALLINT,
    created_by_name TEXT,
    updated_by_name TEXT,
    crew_details JSON
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        ch.crew_hdr_id,
        ch.org_id,
        ch.crew_type_id,
        ch.crew_name,
        ch.driver_id,
        ch.head_count,
        CONCAT(cr.last_name, ', ', cr.first_name) AS created_by_name,
        CONCAT(up.last_name, ', ', up.first_name) AS updated_by_name,
        COALESCE((
            SELECT json_agg(cd)
            FROM public.crew_dtl cd
            WHERE cd.crew_hdr_id = ch.crew_hdr_id
        ), '[]'::json) AS crew_details
    FROM public.crew_hdr ch
    LEFT JOIN public.app_user cr ON cr.user_id = ch.created_by
    LEFT JOIN public.app_user up ON up.user_id = ch.updated_by
    WHERE ch.crew_hdr_id = crew_hdr_id;

    IF NOT FOUND THEN
        RETURN QUERY 
        SELECT 
	        FALSE::BOOLEAN, -- function_status
            'No data found for Crew' || <param2_name>::TEXT, -- msg
	        NULL::BIGINT, -- crew_hdr_id
	        NULL::BIGINT, -- org_id
	        NULL::SMALLINT, -- crew_type_id
	        NULL::CHARACTER VARYING, -- crew_name
	        NULL::BIGINT, -- driver_id
	        NULL::SMALLINT, -- head_count
	        NULL::TEXT, -- created_by_name
	        NULL::TEXT, -- updated_by_name
			'[]'::json; -- crew_details
    END IF;
END;
$$ LANGUAGE plpgsql;

ALTER FUNCTION public.get_crew_hdr_details(bigint) OWNER TO postgres;
GRANT EXECUTE ON FUNCTION public.get_crew_hdr_details(bigint) TO postgres;
```

---
