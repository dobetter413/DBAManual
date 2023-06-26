## Data type conversion reference

Convert data type from Oracle to MySQL

<table>
  <tr><th>Oracle</th><th>MySQL</th><td></td></tr>
  <tr><td>FLOAT</td><td>FLOAT</td><td></td></tr>
  <tr><td>LONG</td><td>LONGTEXT</td><td></td></tr>
  <tr><td>INTEGER</td><td>DECIMAL(38)</td><td></td></tr>
  <tr><td>INT</td><td>DECIMAL(38)</td><td></td></tr>
  <tr><td>SMALLINT</td><td>DECIMAL(38)</td><td></td></tr>
  <tr><td>REAL</td><td>REAL</td><td></td></tr>
  <tr><td>BINARY_DOUBLE</td><td>DOUBLE</td><td></td></tr>
  <tr><td>BINARY_FLOAT</td><td>FLOAT</td><td></td></tr>
  <tr><td>BINARY_INTEGER</td><td>INT</td><td></td></tr>
  <tr><td>DATE</td><td>DATETIME</td><td></td></tr>
  <tr><td>LONG RAW</td><td>LONGBLOB</td><td></td></tr>
  <tr><td>CLOB</td><td>LONGTEXT</td><td></td></tr>
  <tr><td>NCLOB</td><td>LONGTEXT</td><td></td></tr>
  <tr><td>BLOB</td><td>LONGBLOB</td><td></td></tr>
  <tr><td>PLS_INTEGER</td><td>INT</td><td></td></tr>
  <tr><td>NATURAL</td><td>INT</td><td></td></tr>
  <tr><td>NATURALN</td><td>INT</td><td></td></tr>
  <tr><td>POSITIVE</td><td>INT</td><td></td></tr>
  <tr><td>POSITIVEN</td><td>INT</td><td></td></tr>
  <tr><td>SIGNTYPE</td><td>TINYINT</td><td></td></tr>
  <tr><td>SIMPLE_INTEGER</td><td>INT</td><td></td></tr>
  <tr><td>BOOLEAN</td><td>BOOLEAN</td><td></td></tr>
  <tr><td>DECIMAL</td><td>DECIMAL</td><td></td></tr>
  <tr><td>DEC</td><td>DEC</td><td></td></tr>
  <tr><td>NUMERIC</td><td>NUMERIC</td><td></td></tr>
  <tr><td>NVARCHAR2</td><td>VARCHAR</td><td></td></tr>
  <tr><td>VARCHAR</td><td>VARCHAR</td><td></td></tr>
  <tr><td>TIMESTAMP</td><td>DATETIME</td><td></td></tr>
  <tr><td>RAW</td><td>VARBINARY</td><td></td></tr>
  <tr><td>TIMESTAMP WITH TIME ZONE</td><td>DATETIME</td><td></td></tr>
  <tr><td>TIMESTAMP WITH LOCAL TIME ZONE</td><td>DATETIME</td><td></td></tr>
  <tr><td>NUMBER, NUMBER(*)</td><td>DOUBLE</td><td></td></tr>
  <tr><td>NUMBER(*, 0)</td><td>DECIMAL(38)</td><td></td></tr>
  <tr><td>NUMBER(p, s)</td><td>DECIMAL(p, s)</td><td>s > 0</td></tr>
  <tr><td rowspan="5">NUMBER(p, 0), NUMBER(p)</td><td>TINYINT</td><td>1 <= p < 3 </td></tr>
  <tr><td>SMALLINT</td><td>3 <= p < 5</td></tr>
  <tr><td>INT</td><td>5 <= P < 9</td></tr>
  <tr><td>BIGINT</td><td>9 <= p < 19</td></tr>
  <tr><td>DECIMAL(p)</td><td>19 <= 38</td></tr>
  <tr><td rowspan="2">VARCHAR2(n)</td><td>VARCHAR(n)</td><td></td></tr>
  <tr><td>TEXT</td><td>user decide when n > 1000</td></tr>
  <tr><td rowspan="2">CHAR(n), CHARACTER(n)</td><td>CHAR(n)</td><td>1 <= n <= 255</td></tr>
  <tr><td>TEXT</td><td>256 <= n</td></tr>
  <tr><td rowspan="2">NCHAR</td><td>VARCHAR(n)</td><td>1 <= n <= 255</td></tr>
  <tr><td>TEXT</td><td>256 <= n</td></tr>
  <tr><td>ROWID</td><td>Unsupported</td><td></td></tr>
  <tr><td>UROWID</td><td>Unsupported</td><td></td></tr>
  <tr><td>BFILE</td><td>Unsupported</td><td></td></tr>
  <tr><td>INTERVAL YEAR TO MONTH</td><td>Unsupported</td><td></td></tr>
  <tr><td>INTERVAL DAY TO SECOND</td><td>Unsupported</td><td></td></tr>
  <tr><td>ANYTYPE</td><td>Unsupported</td><td></td></tr>
  <tr><td>ANYDATA</td><td>Unsupported</td><td></td></tr>
  <tr><td>ANYDATASET</td><td>Unsupported</td><td></td></tr>
  <tr><td>XMLTYPE</td><td>Unsupported</td><td></td></tr>
  <tr><td>URITYPE</td><td>Unsupported</td><td></td></tr>
  <tr><td>DBURITYPE</td><td>Unsupported</td><td></td></tr>
  <tr><td>XDBURITYPE</td><td>Unsupported</td><td></td></tr>
  <tr><td>HTTPURITYPE</td><td>Unsupported</td><td></td></tr>
  <tr><td>SDO_GEOMETRY</td><td>Unsupported</td><td></td></tr>
  <tr><td>SDO_TOPO_GEOMETRY</td><td>Unsupported</td><td></td></tr>
  <tr><td>SDO_GEORASTER</td><td>Unsupported</td><td></td></tr>
  <tr><td>ORDAUDIO</td><td>Unsupported</td><td></td></tr>
  <tr><td>ORDDICOM</td><td>Unsupported</td><td></td></tr>
  <tr><td>ORDDOC</td><td>Unsupported</td><td></td></tr>
  <tr><td>ORDIMAGE</td><td>Unsupported</td><td></td></tr>
  <tr><td>ORDVIDEO</td><td>Unsupported</td><td></td></tr>
  <tr><td>ORDIMAGESIGNATURE</td><td>Unsupported</td><td></td></tr>
  <tr><td>SI_AVERAGECOLOR</td><td>Unsupported</td><td></td></tr>
  <tr><td>SI_COLOR</td><td>Unsupported</td><td></td></tr>
  <tr><td>SI_COLORHISTOGRAM</td><td>Unsupported</td><td></td></tr>
  <tr><td>SI_FEATURELIST</td><td>Unsupported</td><td></td></tr>
  <tr><td>SI_POSITIONALCOLOR</td><td>Unsupported</td><td></td></tr>
  <tr><td>SI_STILLIMAGE</td><td>Unsupported</td><td></td></tr>
  <tr><td>SI_TEXTURE</td><td>Unsupported</td><td></td></tr>
  <tr><td>EXPRESSION</td><td>Unsupported</td><td></td></tr>
  <tr><td>SYS_REFCURSOR</td><td>Unsupported</td><td></td></tr>
  <tr><td>REFCURSOR</td><td>Unsupported</td><td></td></tr>
  <tr><td>REF</td><td>Unsupported</td><td></td></tr>
</table>

