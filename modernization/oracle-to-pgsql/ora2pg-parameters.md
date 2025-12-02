# Ora2Pg Parameters

Usage: ora2pg [-dhpqv --estimate_cost --dump_as_html] [--option value]

    -a | --allow str  : Comma separated list of objects to allow from export.
                        Can be used with SHOW_COLUMN too.
    -b | --basedir dir: Set the default output directory, where files
                        resulting from exports will be stored.
    -c | --conf file  : Set an alternate configuration file other than the
                        default /etc/ora2pg/ora2pg.conf.
    -C | --cdc_file file: File used to store/read SCN per table during export.
                        default: TABLES_SCN.log in the current directory. This
                        is the file written by the --cdc_ready option.
    -d | --debug      : Enable verbose output.
    -D | --data_type str : Allow custom type replacement at command line.
    -e | --exclude str: Comma separated list of objects to exclude from export.
                        Can be used with SHOW_COLUMN too.
    -h | --help       : Print this short help.
    -g | --grant_object type : Extract privilege from the given object type.
                        See possible values with GRANT_OBJECT configuration.
    -i | --input file : File containing Oracle PL/SQL code to convert with
                        no Oracle database connection initiated.
    -j | --jobs num   : Number of parallel process to send data to PostgreSQL.
    -J | --copies num : Number of parallel connections to extract data from Oracle.
    -l | --log file   : Set a log file. Default is stdout.
    -L | --limit num  : Number of tuples extracted from Oracle and stored in
                        memory before writing, default: 10000.
    -m | --mysql      : Export a MySQL database instead of an Oracle schema.
    -M | --mssql      : Export a Microsoft SQL Server database.
    -n | --namespace schema : Set the Oracle schema to extract from.
    -N | --pg_schema schema : Set PostgreSQL's search_path.
    -o | --out file   : Set the path to the output file where SQL will
                        be written. Default: output.sql in running directory.
    -O | --options    : Used to override any configuration parameter, it can
                        be used multiple time. Syntax: -O "PARAM_NAME=value"
    -p | --plsql      : Enable PLSQL to PLPGSQL code conversion.
    -P | --parallel num: Number of parallel tables to extract at the same time.
    -q | --quiet      : Disable progress bar.
    -r | --relative   : use \ir instead of \i in the psql scripts generated.
    -s | --source DSN : Allow to set the Oracle DBI datasource.
    -S | --scn    SCN : Allow to set the Oracle System Change Number (SCN) to
                        use to export data. It will be used in the WHERE clause
                        to get the data. It is used with action COPY or INSERT.
    -t | --type export: Set the export type. It will override the one
                        given in the configuration file (TYPE).
    -T | --temp_dir dir: Set a distinct temporary directory when two
                        or more ora2pg are run in parallel.
    -u | --user name  : Set the Oracle database connection user.
                        ORA2PG_USER environment variable can be used instead.
    -v | --version    : Show Ora2Pg Version and exit.
    -w | --password pwd : Set the password of the Oracle database user.
                        ORA2PG_PASSWD environment variable can be used instead.
    -W | --where clause : Set the WHERE clause to apply to the Oracle query to
                        retrieve data. Can be used multiple time.
    --forceowner      : Force ora2pg to set tables and sequences owner like in
                  Oracle database. If the value is set to a username this one
                  will be used as the objects owner. By default it's the user
                  used to connect to the Pg database that will be the owner.
    --nls_lang code: Set the Oracle NLS_LANG client encoding.
    --client_encoding code: Set the PostgreSQL client encoding.
    --view_as_table str: Comma separated list of views to export as table.
    --estimate_cost   : Activate the migration cost evaluation with SHOW_REPORT
    --cost_unit_value minutes: Number of minutes for a cost evaluation unit.
                  default: 5 minutes, corresponds to a migration conducted by a
                  PostgreSQL expert. Set it to 10 if this is your first migration.
   --dump_as_html     : Force ora2pg to dump report in HTML, used only with
                        SHOW_REPORT. Default is to dump report as simple text.
   --dump_as_csv      : As above but force ora2pg to dump report in CSV.
   --dump_as_json     : As above but force ora2pg to dump report in JSON.
   --dump_as_sheet    : Report migration assessment with one CSV line per database.
   --dump_as_file_prefix : Filename prefix, suffix will be added depending on
                           dump_as_* selected switches, suffixes
                           will be .html, .csv, .json.
   --init_project name: Initialise a typical ora2pg project tree. Top directory
                        will be created under project base dir.
   --project_base dir : Define the base dir for ora2pg project trees. Default
                        is current directory.
   --print_header     : Used with --dump_as_sheet to print the CSV header
                        especially for the first run of ora2pg.
   --human_days_limit num : Set the number of human-days limit where the migration
                        assessment level switch from B to C. Default is set to
                        5 human-days.
   --audit_user list  : Comma separated list of usernames to filter queries in
                        the DBA_AUDIT_TRAIL table. Used only with SHOW_REPORT
                        and QUERY export type.
   --pg_dsn DSN       : Set the datasource to PostgreSQL for direct import.
   --pg_user name     : Set the PostgreSQL user to use.
   --pg_pwd password  : Set the PostgreSQL password to use.
   --count_rows       : Force ora2pg to perform a real row count in TEST,
                        TEST_COUNT and SHOW_TABLE actions.
   --no_header        : Do not append Ora2Pg header to output file
   --oracle_speed     : Use to know at which speed Oracle is able to send
                        data. No data will be processed or written.
   --ora2pg_speed     : Use to know at which speed Ora2Pg is able to send
                        transformed data. Nothing will be written.
   --blob_to_lo       : export BLOB as large objects, can only be used with
                        action SHOW_COLUMN, TABLE and INSERT.
   --cdc_ready        : use current SCN per table to export data and register
                        them into a file named TABLES_SCN.log per default. It
                        can be changed using -C | --cdc_file.
   --lo_import        : use psql \lo_import command to import BLOB as large
                        object. Can be use to import data with COPY and import
                        large object manually in a second pass. It is recquired
                        for BLOB > 1GB. See documentation for more explanation.
   --mview_as_table str: Comma separated list of materialized views to export
                        as regular table.
   --drop_if_exists   : Drop the object before creation if it exists.
   --delete clause    : Set the DELETE clause to apply to the Oracle query to
                        be applied before importing data. Can be used multiple
                        time.
   --oracle_fdw_prefetch: Set the oracle_fdw prefetch value. Larger values
                        generally result in faster data transfer at the cost
                        of greater memory utilisation at the destination.
    --no_start_scn    : Force Ora2Pg to not use a SCN to export data. By default
                        the current SCN is used to export data from all tables.
    --no_clean_comment: do not try to remove comments in source file before
                        parsing. In some cases it could table a very long time.

See full documentation at [https://ora2pg.darold.net/](https://ora2pg.darold.net/) for more help