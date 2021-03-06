<h1 id="top">🧐🐘<code>fk_arrays_elem_v2.patch</code></h1>

1. [About](#about)
1. [Installation](#installation)
1. [Usage](#usage)
1. [Review](#review)
    1. [doc/src/sgml/catalogs.sgml](#catalogs-sgml)
    1. [doc/src/sgml/ddl.sgml](#ddl-sgml)
    1. [doc/src/sgml/ref/create_table.sgml](#create_table-sgml)
    1. [src/backend/catalog/heap.c](#heap-c)
    1. [src/backend/catalog/pg_constraint.c](#pg_constraint-c)
    1. [src/backend/commands/matview.c](#matview-c)
    1. [src/backend/commands/tablecmds.c](#tablecmds-c)
    1. [src/backend/commands/trigger.c](trigger-c)
    1. [src/backend/commands/typecmds.c](typecmds-c)
    1. [src/backend/nodes/copyfuncs.c](copyfuncs-c)
    1. [src/backend/nodes/equalfuncs.c](equalfuncs-c)
    1. [src/backend/nodes/outfuncs.c](outfuncs-c)
    1. [src/backend/parser/gram.y](gram-y)
    1. [src/backend/parser/parse_utilcmd.c](parse_utilcmd-c)
    1. [src/backend/utils/adt/ri_triggers.c](ri_triggers-c)
    1. [src/backend/utils/adt/ruleutils.c](ruleutils-c)
    1. [src/backend/utils/cache/relcache.c](relcache-c)
    1. [src/include/catalog/pg_constraint.h](pg_constraint-h)
    1. [src/include/nodes/parsenodes.h](parsenodes-h)
    1. [src/include/parser/kwlist.h](kwlist-h)
    1. [src/include/utils/builtins.h](builtins-h)
    1. [src/test/regress/expected/element_fk.out](element_fk-out)
    1. [src/test/regress/parallel_schedule](parallel_schedule)
    1. [src/test/regress/serial_schedule](serial_schedule)
    1. [src/test/regress/sql/element_fk.sql](element_fk-sql)

<h2 id="about">1. About</h2>

This is a review of Mark Rofail's patch [fk_arrays_elem_v2.patch] submitted to the [Commitfest-2021-03].
The [anyarray_anyelement_operators-v4.patch] must be applied before this patch can be tested.

    $ sha512sum fk_arrays_elem_v2.patch
    7edec9bc3132b877247ed677e3f132d12f0759f3745120ad1627a7812608e517f806e22f1ba3464249f43d56e3956af8ed2afbbdaba5731457034d0eafddaaba

[fk_arrays_elem_v2.patch]: https://www.postgresql.org/message-id/attachment/119359/fk_arrays_elem_v2.patch
[anyarray_anyelement_operators-v4.patch]: https://www.postgresql.org/message-id/attachment/119360/anyarray_anyelement_operators-v4.patch
[Commitfest-2021-03]: https://commitfest.postgresql.org/32/2966/

<h2 id="installation">2. Installation</h2>

Patch and compile `PostgreSQL` with:

    $ git clone git://git.postgresql.org/git/postgresql.git
    $ cd postgresql
    $ patch -p1 < ~/Downloads/anyarray_anyelement_operators-v4.patch
    $ patch -p1 < ~/Downloads/fk_arrays_elem_v2.patch
    $ ./configure --prefix=$HOME/pg-head
    $ make -j16
    $ make install
    $ export PATH=$HOME/pg-head/bin:$PATH
    $ initdb -E UTF8 -k -D $HOME/pg-head-data
    $ pg_ctl -D $HOME/pg-head-data -l /tmp/logfile start
    $ createdb $USER
    $ make installcheck

    =======================
    All 202 tests passed.
    =======================

<h2 id="usage">3. Usage</h2>

```sql
CREATE TABLE a (
a_id bigint NOT NULL,
PRIMARY KEY(a_id)
);

CREATE TABLE b_c (
b_id bigint NOT NULL,
c_id bigint NOT NULL,
PRIMARY KEY (b_id, c_id)
);

CREATE TABLE a_b_c (
a_id bigint,
b_id bigint,
c_id bigint[],
FOREIGN KEY (a_id) REFERENCES a,
FOREIGN KEY (b_id, EACH ELEMENT OF c_id) REFERENCES b_c(b_id, c_id)
);

SELECT * FROM unoid.pg_constraint;

                         oid                         |       conname        | connamespace | contype | condeferrable | condeferred | convalidated |      conrelid       | contypid |        conindid        | conparentid |     confrelid     | confupdtype | confdeltype | confmatchtype | conislocal | coninhcount | connoinherit |   conkey    |   confkey   | confreftype |                                                                               conpfeqop                                                                               |                                                                               conppeqop                                                                               |                                                                               conffeqop                                                                               | conexclop | conbin
-----------------------------------------------------+----------------------+--------------+---------+---------------+-------------+--------------+---------------------+----------+------------------------+-------------+-------------------+-------------+-------------+---------------+------------+-------------+--------------+-------------+-------------+-------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------+--------
 [["a", "public"], null, "a_pkey"]                   | a_pkey               | public       | p       | f             | f           | t            | ["a", "public"]     |          | ["a_pkey", "public"]   |             |                   |             |             |               | t          |           0 | t            | {a_id}      |             |             |                                                                                                                                                                       |                                                                                                                                                                       |                                                                                                                                                                       |           |
 [["b_c", "public"], null, "b_c_pkey"]               | b_c_pkey             | public       | p       | f             | f           | t            | ["b_c", "public"]   |          | ["b_c_pkey", "public"] |             |                   |             |             |               | t          |           0 | t            | {b_id,c_id} |             |             |                                                                                                                                                                       |                                                                                                                                                                       |                                                                                                                                                                       |           |
 [["a_b_c", "public"], null, "a_b_c_a_id_fkey"]      | a_b_c_a_id_fkey      | public       | f       | f             | f           | t            | ["a_b_c", "public"] |          | ["a_pkey", "public"]   |             | ["a", "public"]   | a           | a           | s             | t          |           0 | t            | {a_id}      | {a_id}      | {p}         | {"[\"=\", [\"int8\", \"pg_catalog\"], [\"int8\", \"pg_catalog\"], \"pg_catalog\"]"}                                                                                   | {"[\"=\", [\"int8\", \"pg_catalog\"], [\"int8\", \"pg_catalog\"], \"pg_catalog\"]"}                                                                                   | {"[\"=\", [\"int8\", \"pg_catalog\"], [\"int8\", \"pg_catalog\"], \"pg_catalog\"]"}                                                                                   |           |
 [["a_b_c", "public"], null, "a_b_c_b_id_c_id_fkey"] | a_b_c_b_id_c_id_fkey | public       | f       | f             | f           | t            | ["a_b_c", "public"] |          | ["b_c_pkey", "public"] |             | ["b_c", "public"] | a           | a           | s             | t          |           0 | t            | {b_id,c_id} | {b_id,c_id} | {p,e}       | {"[\"=\", [\"int8\", \"pg_catalog\"], [\"int8\", \"pg_catalog\"], \"pg_catalog\"]","[\"=\", [\"int8\", \"pg_catalog\"], [\"int8\", \"pg_catalog\"], \"pg_catalog\"]"} | {"[\"=\", [\"int8\", \"pg_catalog\"], [\"int8\", \"pg_catalog\"], \"pg_catalog\"]","[\"=\", [\"int8\", \"pg_catalog\"], [\"int8\", \"pg_catalog\"], \"pg_catalog\"]"} | {"[\"=\", [\"int8\", \"pg_catalog\"], [\"int8\", \"pg_catalog\"], \"pg_catalog\"]","[\"=\", [\"int8\", \"pg_catalog\"], [\"int8\", \"pg_catalog\"], \"pg_catalog\"]"} |           |
(4 rows)

```

<h2 id="review">4. Review</h2>

<h3 id="catalogs-sgml"><code>doc/src/sgml/catalogs.sgml</code></h3>

```diff
diff --git a/doc/src/sgml/catalogs.sgml b/doc/src/sgml/catalogs.sgml
index db29905e91..6c81596ed0 100644
--- a/doc/src/sgml/catalogs.sgml
+++ b/doc/src/sgml/catalogs.sgml
@@ -2646,6 +2646,16 @@ SCRAM-SHA-256$<replaceable>&lt;iteration count&gt;</replaceable>:<replaceable>&l
       </para></entry>
      </row>
 
+     <row>
+      <entry><structfield>confreftype</structfield></entry>
+      <entry><type>char[]</type></entry>
+      <entry></entry>
+      <entry>If a foreign key, the reference semantics for each column:
+       <literal>p</literal> = plain (simple equality),
+       <literal>e</literal> = each element of referencing array must have a match
+      </entry>
+     </row>
+
      <row>
       <entry role="catalog_table_entry"><para role="column_definition">
        <structfield>conpfeqop</structfield> <type>oid[]</type>
@@ -2701,6 +2711,12 @@ SCRAM-SHA-256$<replaceable>&lt;iteration count&gt;</replaceable>:<replaceable>&l
    </tgroup>
   </table>
 
+  <para>
+    When <structfield>confreftype</structfield> indicates array to scalar
+    foreign key reference semantics, the equality operators listed in
+    <structfield>conpfeqop</structfield> etc are for the array's element type.
```

🧐 Instead of writing "etc", maybe be explicit about what other such columns there are.
It looks like the full list is `conpfeqop`, `conppeqop` and `conffeqop`?

🧐 How it is possible to create an example where you get different values in these arrays,
i.e. when it's not just the type's `=` operator times number of key columns?

```diff
+  </para>
+ 
   <para>
    In the case of an exclusion constraint, <structfield>conkey</structfield>
    is only useful for constraint elements that are simple column references.
```

<h3 id="ddl-sgml"><code>doc/src/sgml/ddl.sgml</code></h3>

```diff
diff --git a/doc/src/sgml/ddl.sgml b/doc/src/sgml/ddl.sgml
index 1e9a4625cc..3f2b69b4f3 100644
--- a/doc/src/sgml/ddl.sgml
+++ b/doc/src/sgml/ddl.sgml
@@ -1079,6 +1079,116 @@ CREATE TABLE order_items (
    </para>
   </sect2>
 
+  <sect2 id="ddl-constraints-element-fk">
+   <title>Array Element Foreign Keys</title>
+
+   <indexterm>
+    <primary>Array Element Foreign Keys</primary>
+   </indexterm>
+
+   <indexterm>
+    <primary>ELEMENT foreign key</primary>
+   </indexterm>
+
+   <indexterm>
+    <primary>constraint</primary>
+    <secondary>Array ELEMENT foreign key</secondary>
+   </indexterm>
+
+   <indexterm>
+    <primary>constraint</primary>
+    <secondary>ELEMENT foreign key</secondary>
+   </indexterm>
+
+   <indexterm>
+    <primary>referential integrity</primary>
+   </indexterm>
+
+   <para>
+    Another option you have with foreign keys is to use a referencing column
+    which is an array of elements with the same type (or a compatible one) as
```

🧐 The "(or a compatible one)" sentence should be removed since the new `<<@` and `@>>`
introduced in the `anyarray_anyelement_operators` patch only supports looking for
an element of the same type as the elements in the array.

```diff
+    the referenced column in the related table. This feature is called
+    <firstterm>Array Element Foreign Keys</firstterm> and is implemented in PostgreSQL
+    with <firstterm>EACH ELEMENT OF foreign key constraints</firstterm>, as
+    described in the following example:
+
+    <programlisting>
+     CREATE TABLE drivers (
+         driver_id integer PRIMARY KEY,
+         first_name text,
+         last_name text
+     );
+
+     CREATE TABLE racing (
+         race_id integer PRIMARY KEY,
+         title text,
+         race_day date,
+         final_positions integer[],
+         FOREIGN KEY <emphasis> (EACH ELEMENT OF final_positions) REFERENCES drivers </emphasis>
+     );
+    </programlisting>
+
+    The above example uses an array (<literal>final_positions</literal>) to
+    store the results of a race: for each of its elements a referential
+    integrity check is enforced on the <literal>drivers</literal> table. Note
+    that <literal>(EACH ELEMENT OF ...) REFERENCES</literal> is an extension of
+    PostgreSQL and it is not included in the SQL standard.
+   </para>
+
+   <para>
+    We currently only support the table constraint form.
+   </para>
+
+   <para>
+    Even though the most common use case for Array Element Foreign Keys constraint is on
+    a single column key, you can define a Array Element Foreign Keys constraint on a
+    group of columns.
+
+    <programlisting>
+     CREATE TABLE available_moves (
+         kind text,
+         move text,
+         description text,
+         PRIMARY KEY (kind, move)
+     );
+
+     CREATE TABLE paths (
+         description text,
+         kind text,
+         moves text[],
+         <emphasis>FOREIGN KEY (kind, EACH ELEMENT OF moves) REFERENCES available_moves (kind, move)</emphasis>
+     );
+
+     INSERT INTO available_moves VALUES ('relative', 'LN', 'look north');
+     INSERT INTO available_moves VALUES ('relative', 'RL', 'rotate left');
+     INSERT INTO available_moves VALUES ('relative', 'RR', 'rotate right');
+     INSERT INTO available_moves VALUES ('relative', 'MF', 'move forward');
+     INSERT INTO available_moves VALUES ('absolute', 'N', 'move north');
+     INSERT INTO available_moves VALUES ('absolute', 'S', 'move south');
+     INSERT INTO available_moves VALUES ('absolute', 'E', 'move east');
+     INSERT INTO available_moves VALUES ('absolute', 'W', 'move west');
+
+     INSERT INTO paths VALUES ('L-shaped path', 'relative', '{LN, RL, MF, RR, MF, MF}');
+     INSERT INTO paths VALUES ('L-shaped path', 'absolute', '{W, N, N}');
+    </programlisting>
+
+    On top of standard foreign key requirements, array
+    <literal>ELEMENT</literal> foreign key constraints require that the
+    referencing column is an array of a compatible type of the corresponding
+    referenced column.
```

🧐 Change "is an array of a compatible type of the corresponding referenced column"
into "is an array with elements of the same type as the corresponding referenced column".

```diff
+   
+    However, it is very important to note that, currently, we only support
+    one array reference per foreign key.
```

Change "However, it is very important to note that, currently, we only support one array reference per foreign key."
into "Only one array reference per foreign key is supported.".

```diff
+   </para>
+
+   <para>
+    For more detailed information on Array Element Foreign Keys options and special
+    cases, please refer to the documentation for 
+    <xref linkend="sql-createtable-foreign-key"/> and
+    <xref linkend="sql-createtable-element-foreign-key-constraints"/>.
+   </para>
+  </sect2>
+
   <sect2 id="ddl-constraints-exclusion">
    <title>Exclusion Constraints</title>
 ```

<h3 id="create_table-sgml"><code>doc/src/sgml/ref/create_table.sgml</code></h3>

```diff
diff --git a/doc/src/sgml/ref/create_table.sgml b/doc/src/sgml/ref/create_table.sgml
index 569f4c9da7..ed5b1b1863 100644
--- a/doc/src/sgml/ref/create_table.sgml
+++ b/doc/src/sgml/ref/create_table.sgml
@@ -1031,10 +1031,10 @@ WITH ( MODULUS <replaceable class="parameter">numeric_literal</replaceable>, REM
     </listitem>
    </varlistentry>
 
-   <varlistentry>
+   <varlistentry id="sql-createtable-foreign-key" xreflabel="FOREIGN KEY">
     <term><literal>REFERENCES <replaceable class="parameter">reftable</replaceable> [ ( <replaceable class="parameter">refcolumn</replaceable> ) ] [ MATCH <replaceable class="parameter">matchtype</replaceable> ] [ ON DELETE <replaceable class="parameter">referential_action</replaceable> ] [ ON UPDATE <replaceable class="parameter">referential_action</replaceable> ]</literal> (column constraint)</term>
 
-   <term><literal>FOREIGN KEY ( <replaceable class="parameter">column_name</replaceable> [, ... ] )
+   <term><literal>FOREIGN KEY ( [EACH ELEMENT OF] <replaceable class="parameter">column_name</replaceable> [, ... ] )
     REFERENCES <replaceable class="parameter">reftable</replaceable> [ ( <replaceable class="parameter">refcolumn</replaceable> [, ... ] ) ]
     [ MATCH <replaceable class="parameter">matchtype</replaceable> ]
     [ ON DELETE <replaceable class="parameter">referential_action</replaceable> ]
@@ -1059,6 +1059,18 @@ WITH ( MODULUS <replaceable class="parameter">numeric_literal</replaceable>, REM
       tables and permanent tables.
      </para>
 
+     <para>
+      In case the column name <replaceable class="parameter">column</replaceable> is 
+      prepended with the <literal>EACH ELEMENT OF</literal> keyword and <replaceable 
+      class="parameter">column</replaceable> is an array of elements compatible with 
```

🧐 Change "is an array of elements compatible with"
into "is an array with elements of the same type as"

```diff
+      the corresponding <replaceable class="parameter">refcolumn</replaceable> in 
+      <replaceable class="parameter">reftable</replaceable>, an Array Element Foreign Key 
+      constraint is put in place (see <xref 
+      linkend="sql-createtable-element-foreign-key-constraints"/> for more 
+      information). Multi-column keys with more than one <literal>EACH ELEMENT 
+      OF</literal> column are currently not allowed. 
+     </para>
+
      <para>
       A value inserted into the referencing column(s) is matched against the
       values of the referenced table and referenced columns using the
@@ -1122,7 +1134,8 @@ WITH ( MODULUS <replaceable class="parameter">numeric_literal</replaceable>, REM
          <para>
           Delete any rows referencing the deleted row, or update the
           values of the referencing column(s) to the new values of the
-          referenced columns, respectively.
+          referenced columns, respectively. 
+          Currently not supported with Array Element Foreign Keys.
          </para>
         </listitem>
        </varlistentry>
@@ -1132,6 +1145,7 @@ WITH ( MODULUS <replaceable class="parameter">numeric_literal</replaceable>, REM
         <listitem>
          <para>
           Set the referencing column(s) to null.
+          Currently not supported with Array Element Foreign Keys.
          </para>
         </listitem>
        </varlistentry>
@@ -1143,6 +1157,7 @@ WITH ( MODULUS <replaceable class="parameter">numeric_literal</replaceable>, REM
           Set the referencing column(s) to their default values.
           (There must be a row in the referenced table matching the default
           values, if they are not null, or the operation will fail.)
+          Currently not supported with Array Element Foreign Keys.
          </para>
         </listitem>
        </varlistentry>
@@ -1158,6 +1173,73 @@ WITH ( MODULUS <replaceable class="parameter">numeric_literal</replaceable>, REM
     </listitem>
    </varlistentry>
 
+   <varlistentry id="sql-createtable-element-foreign-key-constraints" xreflabel="Array ELEMENT Foreign Keys">
+    <term><literal>EACH ELEMENT OF <replaceable class="parameter">reftable</replaceable> [ ( <replaceable class="parameter">refcolumn</replaceable> ) ] [ MATCH <replaceable class="parameter">matchtype</replaceable> ] [ ON DELETE <replaceable class="parameter">action</replaceable> ] [ ON UPDATE <replaceable class="parameter">action</replaceable> ]</literal> (column constraint)</term>
+
+    <listitem>
+     <para>
+      The <literal>EACH ELEMENT OF REFERENCES</literal> definition specifies a 
+      <quote>Array Element Foreign Key</quote>, a special kind of foreign key constraint 
+      requiring the referencing column to be an array of elements of the same type (or 
+      a compatible one) as the referenced column in the referenced table. The value of 
+      each element of the <replaceable class="parameter">refcolumn</replaceable> array 
+      will be matched against some row of <replaceable 
+      class="parameter">reftable</replaceable>. 
+     </para>
+
+     <para>
+      Array Element Foreign Keys are an extension of PostgreSQL and are not included in the SQL standard.
+     </para>
+
+     <para>
+      Even with Array Element Foreign Keys, modifications in the referenced column can trigger 
+      actions to be performed on the referencing array. Similarly to standard foreign 
+      keys, you can specify these actions using the <literal>ON DELETE</literal> and 
+      <literal>ON UPDATE</literal> clauses. However, only the following actions for 
+      each clause are currently allowed:
+
+      <variablelist>
+       <varlistentry>
+        <term><literal>ON UPDATE/DELETE NO ACTION</literal></term>
+        <listitem>
+         <para>
+          Same as standard foreign key constraints. This is the default action.
+         </para>
+        </listitem>
+       </varlistentry>
+
+       <varlistentry>
+        <term><literal>ON UPDATE/DELETE RESTRICT</literal></term>
+        <listitem>
+         <para>
+          Same as standard foreign key constraints.
+         </para>
+        </listitem>
+       </varlistentry>
+      </variablelist>
+     </para>
+
+     <para>
+      It is advisable to index the refrencing column using GIN index as it 
+      considerably enhances the performance. Also concerning coercion while using the 
+      GIN index:
+        
+      <programlisting>
+       CREATE TABLE pktableforarray ( ptest1 int2 PRIMARY KEY, ptest2 text );
+       CREATE TABLE fktableforarray ( ftest1 int4[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY, ftest2 int );
+      </programlisting>
+      This syntax is fine since it will cast ptest1 to int4 upon RI checks,        
+
+      <programlisting>
+       CREATE TABLE pktableforarray ( ptest1 int4 PRIMARY KEY, ptest2 text );
+       CREATE TABLE fktableforarray ( ftest1 int2[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY, ftest2 int );        
+      </programlisting>
+       however, this syntax will cast ftest1 to int4 upon RI checks, thus defeating the
+       purpose of the index.
+     </para>
+    </listitem>
+   </varlistentry>
+
    <varlistentry>
     <term><literal>DEFERRABLE</literal></term>
     <term><literal>NOT DEFERRABLE</literal></term>
@@ -2305,6 +2387,15 @@ CREATE TABLE cities_partdef
    </para>
   </refsect2>
 
+  <refsect2>
+   <title id="sql-createtable-foreign-key-arrays">Array <literal>ELEMENT</literal> Foreign Keys</title>
+
+   <para>
+    Array <literal>ELEMENT</literal> Foreign Keys and the <literal>EACH ELEMENT 
+    OF</literal> clause are a <productname>PostgreSQL</productname> extension. 
+   </para>
+  </refsect2>
+
   <refsect2>
    <title><literal>PARTITION BY</literal> Clause</title>
 ```

<h3 id="heap-c"><code>src/backend/catalog/heap.c</code></h3>

```diff
diff --git a/src/backend/catalog/heap.c b/src/backend/catalog/heap.c
index 9abc4a1f55..a9e2ab33b0 100644
--- a/src/backend/catalog/heap.c
+++ b/src/backend/catalog/heap.c
@@ -2463,6 +2463,7 @@ StoreRelCheck(Relation rel, const char *ccname, Node *expr,
 							  NULL,
 							  NULL,
 							  NULL,
+							  NULL,
 							  0,
 							  ' ',
 							  ' ',
```

<h3 id="index-c"><code>src/backend/catalog/index.c</code></h3>

```diff
diff --git a/src/backend/catalog/index.c b/src/backend/catalog/index.c
index 1514937748..48c6d3aaa8 100644
--- a/src/backend/catalog/index.c
+++ b/src/backend/catalog/index.c
@@ -2109,6 +2109,7 @@ index_constraint_create(Relation heapRelation,
 								   NULL,
 								   NULL,
 								   NULL,
+								   NULL,
```

The positions up until the first `NULL` are nicely commented,
but not the ones with `NULL`, `0` or `' '`, even though
these are probably the ones in most need of a comment.
This is not your fault since the other not-commented values
were there before, but maybe now is a good time to give them
comments as well to increase readability.

```diff
 								   0,
 								   ' ',
 								   ' ',
```

<h3 id="pg_constraint-c"><code>src/backend/catalog/pg_constraint.c</code></h3>

```diff
diff --git a/src/backend/catalog/pg_constraint.c b/src/backend/catalog/pg_constraint.c
index 0081558c48..be1fc54a35 100644
--- a/src/backend/catalog/pg_constraint.c
+++ b/src/backend/catalog/pg_constraint.c
@@ -62,6 +62,7 @@ CreateConstraintEntry(const char *constraintName,
 					  Oid indexRelId,
 					  Oid foreignRelId,
 					  const int16 *foreignKey,
+					  const char *foreignRefType,
 					  const Oid *pfEqOp,
 					  const Oid *ppEqOp,
 					  const Oid *ffEqOp,
@@ -84,6 +85,7 @@ CreateConstraintEntry(const char *constraintName,
 	Datum		values[Natts_pg_constraint];
 	ArrayType  *conkeyArray;
 	ArrayType  *confkeyArray;
+	ArrayType  *confreftypeArray;
 	ArrayType  *conpfeqopArray;
 	ArrayType  *conppeqopArray;
 	ArrayType  *conffeqopArray;
@@ -123,7 +125,11 @@ CreateConstraintEntry(const char *constraintName,
 		for (i = 0; i < foreignNKeys; i++)
 			fkdatums[i] = Int16GetDatum(foreignKey[i]);
 		confkeyArray = construct_array(fkdatums, foreignNKeys,
-									   INT2OID, 2, true, TYPALIGN_SHORT);
+									   INT2OID, sizeof(int16), true, TYPALIGN_SHORT);
+		for (i = 0; i < foreignNKeys; i++)
+			fkdatums[i] = CharGetDatum(foreignRefType[i]);
+		confreftypeArray = construct_array(fkdatums, foreignNKeys,
+										   CHAROID, sizeof(char), true, TYPALIGN_CHAR);
 		for (i = 0; i < foreignNKeys; i++)
 			fkdatums[i] = ObjectIdGetDatum(pfEqOp[i]);
 		conpfeqopArray = construct_array(fkdatums, foreignNKeys,
@@ -140,6 +146,7 @@ CreateConstraintEntry(const char *constraintName,
 	else
 	{
 		confkeyArray = NULL;
+		confreftypeArray = NULL;
 		conpfeqopArray = NULL;
 		conppeqopArray = NULL;
 		conffeqopArray = NULL;
@@ -196,6 +203,11 @@ CreateConstraintEntry(const char *constraintName,
 	else
 		nulls[Anum_pg_constraint_confkey - 1] = true;
 
+	if (confreftypeArray)
+		values[Anum_pg_constraint_confreftype - 1] = PointerGetDatum(confreftypeArray);
+	else
+		nulls[Anum_pg_constraint_confreftype - 1] = true;
+
 	if (conpfeqopArray)
 		values[Anum_pg_constraint_conpfeqop - 1] = PointerGetDatum(conpfeqopArray);
 	else
@@ -1163,7 +1175,8 @@ get_primary_key_attnos(Oid relid, bool deferrableOk, Oid *constraintOid)
 void
 DeconstructFkConstraintRow(HeapTuple tuple, int *numfks,
 						   AttrNumber *conkey, AttrNumber *confkey,
-						   Oid *pf_eq_oprs, Oid *pp_eq_oprs, Oid *ff_eq_oprs)
+						   Oid *pf_eq_oprs, Oid *pp_eq_oprs, Oid *ff_eq_oprs,
+						   char *fk_reftypes)
 {
 	Oid			constrId;
 	Datum		adatum;
@@ -1208,6 +1221,25 @@ DeconstructFkConstraintRow(HeapTuple tuple, int *numfks,
 	if ((Pointer) arr != DatumGetPointer(adatum))
 		pfree(arr);				/* free de-toasted copy, if any */
 
+	if (fk_reftypes)
+	{
+		adatum = SysCacheGetAttr(CONSTROID, tuple,
+								 Anum_pg_constraint_confreftype, &isNull);
+		if (isNull)
+			elog(ERROR, "null confreftype for constraint %u", constrId);
+		arr = DatumGetArrayTypeP(adatum);	/* ensure not toasted */
+		if (ARR_NDIM(arr) != 1 ||
+			ARR_ELEMTYPE(arr) != CHAROID)
+			elog(ERROR, "confreftype is not a 1-D char array");
+		if (ARR_HASNULL(arr))
+			elog(ERROR, "confreftype contains nulls");
+		if (ARR_DIMS(arr)[0] != numkeys)
+			elog(ERROR, "confreftype length does not equal the number of foreign keys");
+		memcpy(fk_reftypes, ARR_DATA_PTR(arr), numkeys * sizeof(char));
+		if ((Pointer) arr != DatumGetPointer(adatum))
+			pfree(arr);				/* free de-toasted copy, if any */
+	}
+
 	if (pf_eq_oprs)
 	{
 		adatum = SysCacheGetAttr(CONSTROID, tuple,
```

<h3 id="matview-c"><code>src/backend/commands/matview.c</code></h3>

```diff
diff --git a/src/backend/commands/matview.c b/src/backend/commands/matview.c
index c5c25ce11d..856b2b6775 100644
--- a/src/backend/commands/matview.c
+++ b/src/backend/commands/matview.c
@@ -764,7 +764,8 @@ refresh_by_match_merge(Oid matviewOid, Oid tempOid, Oid relowner,
 				generate_operator_clause(&querybuf,
 										 leftop, attrtype,
 										 op,
-										 rightop, attrtype);
+										 rightop, attrtype,
+										 FKCONSTR_REF_PLAIN);
 
 				foundUniqueIndex = true;
 			}
```

<h3 id="tablecmds-c"><code>src/backend/commands/tablecmds.c</code></h3>

```diff
diff --git a/src/backend/commands/tablecmds.c b/src/backend/commands/tablecmds.c
index 420991e315..e2a8f2791c 100644
--- a/src/backend/commands/tablecmds.c
+++ b/src/backend/commands/tablecmds.c
@@ -447,12 +447,12 @@ static ObjectAddress ATAddForeignKeyConstraint(List **wqueue, AlteredTableInfo *
 											   LOCKMODE lockmode);
 static ObjectAddress addFkRecurseReferenced(List **wqueue, Constraint *fkconstraint,
 											Relation rel, Relation pkrel, Oid indexOid, Oid parentConstr,
-											int numfks, int16 *pkattnum, int16 *fkattnum,
+											int numfks, int16 *pkattnum, int16 *fkattnum, char *fkreftypes,
 											Oid *pfeqoperators, Oid *ppeqoperators, Oid *ffeqoperators,
 											bool old_check_ok);
-static void addFkRecurseReferencing(List **wqueue, Constraint *fkconstraint,
-									Relation rel, Relation pkrel, Oid indexOid, Oid parentConstr,
-									int numfks, int16 *pkattnum, int16 *fkattnum,
+static void addFkRecurseReferencing(List **wqueue, Constraint *fkconstraint, Relation rel,
+									Relation pkrel, Oid indexOid, Oid parentConstr,
+									int numfks, int16 *pkattnum, int16 *fkattnum, char *fkreftypes,
 									Oid *pfeqoperators, Oid *ppeqoperators, Oid *ffeqoperators,
 									bool old_check_ok, LOCKMODE lockmode);
 static void CloneForeignKeyConstraints(List **wqueue, Relation parentRel,
@@ -8486,6 +8486,7 @@ ATAddForeignKeyConstraint(List **wqueue, AlteredTableInfo *tab, Relation rel,
 	Relation	pkrel;
 	int16		pkattnum[INDEX_MAX_KEYS];
 	int16		fkattnum[INDEX_MAX_KEYS];
+	char		fkreftypes[INDEX_MAX_KEYS];
 	Oid			pktypoid[INDEX_MAX_KEYS];
 	Oid			fktypoid[INDEX_MAX_KEYS];
 	Oid			opclasses[INDEX_MAX_KEYS];
@@ -8493,9 +8494,11 @@ ATAddForeignKeyConstraint(List **wqueue, AlteredTableInfo *tab, Relation rel,
 	Oid			ppeqoperators[INDEX_MAX_KEYS];
 	Oid			ffeqoperators[INDEX_MAX_KEYS];
 	int			i;
+	ListCell   *lc;
 	int			numfks,
 				numpks;
 	Oid			indexOid;
+	bool		has_array;
 	bool		old_check_ok;
 	ObjectAddress address;
 	ListCell   *old_pfeqop_item = list_head(fkconstraint->old_conpfeqop);
@@ -8584,6 +8587,7 @@ ATAddForeignKeyConstraint(List **wqueue, AlteredTableInfo *tab, Relation rel,
 	 */
 	MemSet(pkattnum, 0, sizeof(pkattnum));
 	MemSet(fkattnum, 0, sizeof(fkattnum));
+	MemSet(fkreftypes, 0, sizeof(fkreftypes));
 	MemSet(pktypoid, 0, sizeof(pktypoid));
 	MemSet(fktypoid, 0, sizeof(fktypoid));
 	MemSet(opclasses, 0, sizeof(opclasses));
@@ -8595,6 +8599,54 @@ ATAddForeignKeyConstraint(List **wqueue, AlteredTableInfo *tab, Relation rel,
 									 fkconstraint->fk_attrs,
 									 fkattnum, fktypoid);
 
+	/*
+	 * Validate the reference semantics codes, too, and convert list to array
+	 * format to pass to CreateConstraintEntry.
+	 */
+	Assert(list_length(fkconstraint->fk_reftypes) == numfks);
+	has_array = false;
+	i = 0;
+	foreach(lc, fkconstraint->fk_reftypes)
+	{
+		char		reftype = lfirst_int(lc);
+
+		switch (reftype)
+		{
+			case FKCONSTR_REF_PLAIN:
+				/* OK, nothing to do */
+				break;
+			case FKCONSTR_REF_EACH_ELEMENT:
+				/* At most one FK column can be an array reference */
+				if (has_array)
+					ereport(ERROR,
+							(errcode(ERRCODE_INVALID_FOREIGN_KEY),
+							 errmsg("foreign keys support only one array column")));
+
+				has_array = true;
+				break;
+			default:
+				elog(ERROR, "invalid fk_reftype: %d", (int) reftype);
+				break;
+		}
+		fkreftypes[i] = reftype;
+		i++;
+	}
+
+	/*
+	* Array foreign keys support only UPDATE/DELETE NO ACTION, UPDATE/DELETE
+	* RESTRICT
+	*/
+	if (has_array)
+	{
+		if ((fkconstraint->fk_upd_action != FKCONSTR_ACTION_NOACTION &&
+			 fkconstraint->fk_upd_action != FKCONSTR_ACTION_RESTRICT) ||
+			(fkconstraint->fk_del_action != FKCONSTR_ACTION_NOACTION &&
+			 fkconstraint->fk_del_action != FKCONSTR_ACTION_RESTRICT))
+			ereport(ERROR,
+					(errcode(ERRCODE_INVALID_FOREIGN_KEY),
+					 errmsg("Array Element Foreign Keys support only NO ACTION and RESTRICT actions")));
+	}
+
 	/*
 	 * If the attribute list for the referenced table was omitted, lookup the
 	 * definition of the primary key and use it.  Otherwise, validate the
@@ -8708,6 +8760,78 @@ ATAddForeignKeyConstraint(List **wqueue, AlteredTableInfo *tab, Relation rel,
 			elog(ERROR, "only b-tree indexes are supported for foreign keys");
 		eqstrategy = BTEqualStrategyNumber;
 
+		/*
+		 * If this is an array foreign key, we must look up the operators for
+		 * the array element type, not the array type itself.
+		 */
+		if (fkreftypes[i] == FKCONSTR_REF_EACH_ELEMENT)
+		{
+			Oid			elemopclass;
+
+			/* We check if the array element type exists and is of valid Oid */
+			fktype = get_base_element_type(fktype);
+			if (!OidIsValid(fktype))
+				ereport(ERROR,
+						(errcode(ERRCODE_INVALID_FOREIGN_KEY),
+						 errmsg("foreign key constraint \"%s\" cannot be implemented",
+								fkconstraint->conname),
+						 errdetail("Key column \"%s\" has type %s which is not an array type.",
+								   strVal(list_nth(fkconstraint->fk_attrs, i)),
+								   format_type_be(fktypoid[i]))));
+
+			/*
+			 * make sure the <<@ operator can work with the specified operand
+			 * types
+			 */
+			if (fktype != pktype)
+				ereport(ERROR,
+				(errcode(ERRCODE_INVALID_FOREIGN_KEY),
+					errmsg("foreign key constraint \"%s\" cannot be implemented",
+						fkconstraint->conname),
+					errdetail("Unssupported relation between %s and %s.",
+							format_type_be(fktypoid[i]),
+							format_type_be(pktype))));
+
+			/*
+			 * For the moment, we must also insist that the array's element
+			 * type have a default btree opclass that is in the index's
+			 * opfamily.  This is necessary because ri_triggers.c relies on
+			 * COUNT(DISTINCT x) on the element type, as well as on array_eq()
+			 * on the array type, and we need those operations to have the
+			 * same notion of equality that we're using otherwise.
+			 *
+			 * XXX this restriction is pretty annoying, considering the effort
+			 * that's been put into the rest of the RI mechanisms to make them
+			 * work with nondefault equality operators.  In particular, it
+			 * means that the cast-to-PK-datatype code path isn't useful for
+			 * array-to-scalar references.
+			 */
+			elemopclass = GetDefaultOpClass(fktype, BTREE_AM_OID);
+			if (!OidIsValid(elemopclass) ||
+				get_opclass_family(elemopclass) != opfamily)
+			{
+				/* Get the index opclass's name for the error message. */
+				char	   *opcname;
+
+				cla_ht = SearchSysCache1(CLAOID,
+										 ObjectIdGetDatum(opclasses[i]));
+				if (!HeapTupleIsValid(cla_ht))
+					elog(ERROR, "cache lookup failed for opclass %u",
+						 opclasses[i]);
+				cla_tup = (Form_pg_opclass) GETSTRUCT(cla_ht);
+				opcname = pstrdup(NameStr(cla_tup->opcname));
+				ReleaseSysCache(cla_ht);
+				ereport(ERROR,
+						(errcode(ERRCODE_DATATYPE_MISMATCH),
+						 errmsg("foreign key constraint \"%s\" cannot be implemented",
+								fkconstraint->conname),
+						 errdetail("Key column \"%s\" has element type %s which does not have a default btree operator class that's compatible with class \"%s\".",
+								   strVal(list_nth(fkconstraint->fk_attrs, i)),
+								   format_type_be(fktype),
+								   opcname)));
+			}
+		}
+
 		/*
 		 * There had better be a primary equality operator for the index.
 		 * We'll use it for PK = PK comparisons.
@@ -8807,6 +8931,13 @@ ATAddForeignKeyConstraint(List **wqueue, AlteredTableInfo *tab, Relation rel,
 			 * We may assume that pg_constraint.conkey is not changing.
 			 */
 			old_fktype = attr->atttypid;
+			if (fkreftypes[i] == FKCONSTR_REF_EACH_ELEMENT)
+			{
+				old_fktype = get_base_element_type(old_fktype);
+				/* this shouldn't happen ... */
+				if (!OidIsValid(old_fktype))
+					elog(ERROR, "old foreign key column is not an array");
+			}
 			new_fktype = fktype;
 			old_pathtype = findFkeyCast(pfeqop_right, old_fktype,
 										&old_castfunc);
@@ -8866,6 +8997,7 @@ ATAddForeignKeyConstraint(List **wqueue, AlteredTableInfo *tab, Relation rel,
 									 numfks,
 									 pkattnum,
 									 fkattnum,
+									 fkreftypes,
 									 pfeqoperators,
 									 ppeqoperators,
 									 ffeqoperators,
@@ -8878,6 +9010,7 @@ ATAddForeignKeyConstraint(List **wqueue, AlteredTableInfo *tab, Relation rel,
 							numfks,
 							pkattnum,
 							fkattnum,
+							fkreftypes,
 							pfeqoperators,
 							ppeqoperators,
 							ffeqoperators,
@@ -8921,7 +9054,7 @@ static ObjectAddress
 addFkRecurseReferenced(List **wqueue, Constraint *fkconstraint, Relation rel,
 					   Relation pkrel, Oid indexOid, Oid parentConstr,
 					   int numfks,
-					   int16 *pkattnum, int16 *fkattnum, Oid *pfeqoperators,
+					   int16 *pkattnum, int16 *fkattnum, char *fkreftypes, Oid *pfeqoperators,
 					   Oid *ppeqoperators, Oid *ffeqoperators, bool old_check_ok)
 {
 	ObjectAddress address;
@@ -8991,6 +9124,7 @@ addFkRecurseReferenced(List **wqueue, Constraint *fkconstraint, Relation rel,
 									  indexOid,
 									  RelationGetRelid(pkrel),
 									  pkattnum,
+									  fkreftypes,
 									  pfeqoperators,
 									  ppeqoperators,
 									  ffeqoperators,
@@ -9076,7 +9210,7 @@ addFkRecurseReferenced(List **wqueue, Constraint *fkconstraint, Relation rel,
 					 indexOid, RelationGetRelationName(partRel));
 			addFkRecurseReferenced(wqueue, fkconstraint, rel, partRel,
 								   partIndexId, constrOid, numfks,
-								   mapped_pkattnum, fkattnum,
+								   mapped_pkattnum, fkattnum, fkreftypes,
 								   pfeqoperators, ppeqoperators, ffeqoperators,
 								   old_check_ok);
 
@@ -9125,7 +9259,7 @@ addFkRecurseReferenced(List **wqueue, Constraint *fkconstraint, Relation rel,
 static void
 addFkRecurseReferencing(List **wqueue, Constraint *fkconstraint, Relation rel,
 						Relation pkrel, Oid indexOid, Oid parentConstr,
-						int numfks, int16 *pkattnum, int16 *fkattnum,
+						int numfks, int16 *pkattnum, int16 *fkattnum, char *fkreftypes,
 						Oid *pfeqoperators, Oid *ppeqoperators, Oid *ffeqoperators,
 						bool old_check_ok, LOCKMODE lockmode)
 {
@@ -9259,6 +9393,7 @@ addFkRecurseReferencing(List **wqueue, Constraint *fkconstraint, Relation rel,
 									  indexOid,
 									  RelationGetRelid(pkrel),
 									  pkattnum,
+									  fkreftypes,
 									  pfeqoperators,
 									  ppeqoperators,
 									  ffeqoperators,
@@ -9294,6 +9429,7 @@ addFkRecurseReferencing(List **wqueue, Constraint *fkconstraint, Relation rel,
 									numfks,
 									pkattnum,
 									mapped_fkattnum,
+									fkreftypes,
 									pfeqoperators,
 									ppeqoperators,
 									ffeqoperators,
@@ -9402,6 +9538,7 @@ CloneFkReferenced(Relation parentRel, Relation partitionRel)
 		Oid			conpfeqop[INDEX_MAX_KEYS];
 		Oid			conppeqop[INDEX_MAX_KEYS];
 		Oid			conffeqop[INDEX_MAX_KEYS];
+		char		confreftype[INDEX_MAX_KEYS];
 		Constraint *fkconstraint;
 
 		tuple = SearchSysCache1(CONSTROID, constrOid);
@@ -9434,8 +9571,8 @@ CloneFkReferenced(Relation parentRel, Relation partitionRel)
 								   confkey,
 								   conpfeqop,
 								   conppeqop,
-								   conffeqop);
-
+								   conffeqop,
+								   confreftype);
 		for (int i = 0; i < numfks; i++)
 			mapped_confkey[i] = attmap->attnums[confkey[i] - 1];
 
@@ -9478,6 +9615,7 @@ CloneFkReferenced(Relation parentRel, Relation partitionRel)
 							   numfks,
 							   mapped_confkey,
 							   conkey,
+							   confreftype,
 							   conpfeqop,
 							   conppeqop,
 							   conffeqop,
@@ -9548,6 +9686,7 @@ CloneFkReferencing(List **wqueue, Relation parentRel, Relation partRel)
 		AttrNumber	conkey[INDEX_MAX_KEYS];
 		AttrNumber	mapped_conkey[INDEX_MAX_KEYS];
 		AttrNumber	confkey[INDEX_MAX_KEYS];
+		char		confreftype[INDEX_MAX_KEYS];
 		Oid			conpfeqop[INDEX_MAX_KEYS];
 		Oid			conppeqop[INDEX_MAX_KEYS];
 		Oid			conffeqop[INDEX_MAX_KEYS];
@@ -9582,7 +9721,8 @@ CloneFkReferencing(List **wqueue, Relation parentRel, Relation partRel)
 									   ShareRowExclusiveLock, NULL);
 
 		DeconstructFkConstraintRow(tuple, &numfks, conkey, confkey,
-								   conpfeqop, conppeqop, conffeqop);
+								   conpfeqop, conppeqop, conffeqop,
+								   confreftype);
 		for (int i = 0; i < numfks; i++)
 			mapped_conkey[i] = attmap->attnums[conkey[i] - 1];
 
@@ -9661,6 +9801,7 @@ CloneFkReferencing(List **wqueue, Relation parentRel, Relation partRel)
 								  indexOid,
 								  constrForm->confrelid,	/* same foreign rel */
 								  confkey,
+								  confreftype,
 								  conpfeqop,
 								  conppeqop,
 								  conffeqop,
@@ -9699,6 +9840,7 @@ CloneFkReferencing(List **wqueue, Relation parentRel, Relation partRel)
 								numfks,
 								confkey,
 								mapped_conkey,
+								confreftype,
 								conpfeqop,
 								conppeqop,
 								conffeqop,
```

<h3 id="trigger-c"><code>src/backend/commands/trigger.c</code></h3>

```diff
diff --git a/src/backend/commands/trigger.c b/src/backend/commands/trigger.c
index 8908847c6c..5ced45a857 100644
--- a/src/backend/commands/trigger.c
+++ b/src/backend/commands/trigger.c
@@ -801,6 +801,7 @@ CreateTrigger(CreateTrigStmt *stmt, const char *queryString,
 											  NULL,
 											  NULL,
 											  NULL,
+											  NULL,
 											  0,
 											  ' ',
 											  ' ',
```

<h3 id="typecmds-c"><code>src/backend/commands/typecmds.c</code></h3>

```diff
diff --git a/src/backend/commands/typecmds.c b/src/backend/commands/typecmds.c
index 76218fb47e..3fd3f0fe57 100644
--- a/src/backend/commands/typecmds.c
+++ b/src/backend/commands/typecmds.c
@@ -3553,6 +3553,7 @@ domainAddConstraint(Oid domainOid, Oid domainNamespace, Oid baseTypeOid,
 							  NULL,
 							  NULL,
 							  NULL,
+							  NULL,
 							  0,
 							  ' ',
 							  ' ',
```

<h3 id="copyfuncs-c"><code>src/backend/nodes/copyfuncs.c</code></h3>

```diff
diff --git a/src/backend/nodes/copyfuncs.c b/src/backend/nodes/copyfuncs.c
index 65bbc18ecb..0d5bf91439 100644
--- a/src/backend/nodes/copyfuncs.c
+++ b/src/backend/nodes/copyfuncs.c
@@ -3011,6 +3011,7 @@ _copyConstraint(const Constraint *from)
 	COPY_NODE_FIELD(pktable);
 	COPY_NODE_FIELD(fk_attrs);
 	COPY_NODE_FIELD(pk_attrs);
+	COPY_NODE_FIELD(fk_reftypes);
 	COPY_SCALAR_FIELD(fk_matchtype);
 	COPY_SCALAR_FIELD(fk_upd_action);
 	COPY_SCALAR_FIELD(fk_del_action);
```

<h3 id="equalfuncs-c"><code>src/backend/nodes/equalfuncs.c</code></h3>

```diff
diff --git a/src/backend/nodes/equalfuncs.c b/src/backend/nodes/equalfuncs.c
index c2d73626fc..67e001d822 100644
--- a/src/backend/nodes/equalfuncs.c
+++ b/src/backend/nodes/equalfuncs.c
@@ -2642,6 +2642,7 @@ _equalConstraint(const Constraint *a, const Constraint *b)
 	COMPARE_NODE_FIELD(pktable);
 	COMPARE_NODE_FIELD(fk_attrs);
 	COMPARE_NODE_FIELD(pk_attrs);
+	COMPARE_NODE_FIELD(fk_reftypes);
 	COMPARE_SCALAR_FIELD(fk_matchtype);
 	COMPARE_SCALAR_FIELD(fk_upd_action);
 	COMPARE_SCALAR_FIELD(fk_del_action);
```

<h3 id="outfuncs-c"><code>src/backend/nodes/outfuncs.c</code></h3>

```diff
diff --git a/src/backend/nodes/outfuncs.c b/src/backend/nodes/outfuncs.c
index f5dcedf6e8..0c582bb3f7 100644
--- a/src/backend/nodes/outfuncs.c
+++ b/src/backend/nodes/outfuncs.c
@@ -3630,6 +3630,7 @@ _outConstraint(StringInfo str, const Constraint *node)
 			WRITE_NODE_FIELD(pktable);
 			WRITE_NODE_FIELD(fk_attrs);
 			WRITE_NODE_FIELD(pk_attrs);
+			WRITE_NODE_FIELD(fk_reftypes);
 			WRITE_CHAR_FIELD(fk_matchtype);
 			WRITE_CHAR_FIELD(fk_upd_action);
 			WRITE_CHAR_FIELD(fk_del_action);
```

<h3 id="gram-y"><code>src/backend/parser/gram.y</code></h3>

```diff
diff --git a/src/backend/parser/gram.y b/src/backend/parser/gram.y
index dd72a9fc3c..6655a617db 100644
--- a/src/backend/parser/gram.y
+++ b/src/backend/parser/gram.y
@@ -134,6 +134,19 @@ typedef struct SelectLimit
 	LimitOption limitOption;
 } SelectLimit;
 
+/*
+ * Private struct for the result of foreign_key_column_elem production contains
+ * reftype that denotes type of foreign key; can be oen of the following:
+ *
+ * FKCONSTR_REF_PLAIN 			plain foreign keys
+ * FKCONSTR_REF_EACH_ELEMENT	foreign key arrays
+ */
+typedef struct FKColElem
+{
+	Node	   *name;			/* name of the column (a String) */
+	char		reftype;		/* FKCONSTR_REF_xxx code */
+} FKColElem;
+
 /* ConstraintAttributeSpec yields an integer bitmask of these flags: */
 #define CAS_NOT_DEFERRABLE			0x01
 #define CAS_DEFERRABLE				0x02
@@ -191,6 +204,7 @@ static RangeVar *makeRangeVarFromAnyName(List *names, int position, core_yyscan_
 static void SplitColQualList(List *qualList,
 							 List **constraintList, CollateClause **collClause,
 							 core_yyscan_t yyscanner);
+static void SplitFKColElems(List *fkcolelems, List **names, List **reftypes);
 static void processCASbits(int cas_bits, int location, const char *constrType,
 			   bool *deferrable, bool *initdeferred, bool *not_valid,
 			   bool *no_inherit, core_yyscan_t yyscanner);
@@ -241,6 +255,7 @@ static Node *makeRecursiveViewSelect(char *relname, List *aliases, Node *query);
 	A_Indices			*aind;
 	ResTarget			*target;
 	struct PrivTarget	*privtarget;
+	struct FKColElem	*fkcolelem;
 	AccessPriv			*accesspriv;
 	struct ImportQual	*importqual;
 	InsertStmt			*istmt;
@@ -375,6 +390,7 @@ static Node *makeRecursiveViewSelect(char *relname, List *aliases, Node *query);
 %type <accesspriv> privilege
 %type <list>	privileges privilege_list
 %type <privtarget> privilege_target
+%type <fkcolelem> foreign_key_column_elem
 %type <objwithargs> function_with_argtypes aggregate_with_argtypes operator_with_argtypes procedure_with_argtypes function_with_argtypes_common
 %type <list>	function_with_argtypes_list aggregate_with_argtypes_list operator_with_argtypes_list procedure_with_argtypes_list
 %type <ival>	defacl_privilege_target
@@ -412,7 +428,7 @@ static Node *makeRecursiveViewSelect(char *relname, List *aliases, Node *query);
 				execute_param_clause using_clause returning_clause
 				opt_enum_val_list enum_val_list table_func_column_list
 				create_generic_options alter_generic_options
-				relation_expr_list dostmt_opt_list
+				relation_expr_list dostmt_opt_list foreign_key_column_list
 				transform_element_list transform_type_list
 				TriggerTransitions TriggerReferencing
 				vacuum_relation_list opt_vacuum_relation_list
@@ -642,7 +658,7 @@ static Node *makeRecursiveViewSelect(char *relname, List *aliases, Node *query);
 	DETACH DICTIONARY DISABLE_P DISCARD DISTINCT DO DOCUMENT_P DOMAIN_P
 	DOUBLE_P DROP
 
-	EACH ELSE ENABLE_P ENCODING ENCRYPTED END_P ENUM_P ESCAPE EVENT EXCEPT
+	EACH ELEMENT ELSE ENABLE_P ENCODING ENCRYPTED END_P ENUM_P ESCAPE EVENT EXCEPT
 	EXCLUDE EXCLUDING EXCLUSIVE EXECUTE EXISTS EXPLAIN EXPRESSION
 	EXTENSION EXTERNAL EXTRACT
 
@@ -3621,8 +3637,10 @@ ColConstraintElem:
 					n->contype = CONSTR_FOREIGN;
 					n->location = @1;
 					n->pktable			= $2;
+					/* fk_attrs will be filled in by parse analysis */
 					n->fk_attrs			= NIL;
 					n->pk_attrs			= $3;
+					n->fk_reftypes		= list_make1_int(FKCONSTR_REF_PLAIN);
 					n->fk_matchtype		= $4;
 					n->fk_upd_action	= (char) ($5 >> 8);
 					n->fk_del_action	= (char) ($5 & 0xFF);
@@ -3824,14 +3842,15 @@ ConstraintElem:
 								   NULL, yyscanner);
 					$$ = (Node *)n;
 				}
-			| FOREIGN KEY '(' columnList ')' REFERENCES qualified_name
-				opt_column_list key_match key_actions ConstraintAttributeSpec
+			| FOREIGN KEY '(' foreign_key_column_list ')' REFERENCES
+				qualified_name opt_column_list key_match key_actions
+				ConstraintAttributeSpec
 				{
 					Constraint *n = makeNode(Constraint);
 					n->contype = CONSTR_FOREIGN;
 					n->location = @1;
+					SplitFKColElems($4, &n->fk_attrs, &n->fk_reftypes);
 					n->pktable			= $7;
-					n->fk_attrs			= $4;
 					n->pk_attrs			= $8;
 					n->fk_matchtype		= $9;
 					n->fk_upd_action	= (char) ($10 >> 8);
@@ -3865,6 +3884,30 @@ columnElem: ColId
 				}
 		;
 
+foreign_key_column_list:
+			foreign_key_column_elem
+				{ $$ = list_make1($1); }
+			| foreign_key_column_list ',' foreign_key_column_elem
+				{ $$ = lappend($1, $3); }
+		;
+
+foreign_key_column_elem:
+			ColId
+				{
+					FKColElem *n = (FKColElem *) palloc(sizeof(FKColElem));
+					n->name = (Node *) makeString($1);
+					n->reftype = FKCONSTR_REF_PLAIN;
+					$$ = n;
+				}
+			| EACH ELEMENT OF ColId
+				{
+					FKColElem *n = (FKColElem *) palloc(sizeof(FKColElem));
+					n->name = (Node *) makeString($4);
+					n->reftype = FKCONSTR_REF_EACH_ELEMENT;
+					$$ = n;
+				}
+		;
+
 opt_c_include:	INCLUDE '(' columnList ')'			{ $$ = $3; }
 			 |		/* EMPTY */						{ $$ = NIL; }
 		;
@@ -15321,6 +15364,7 @@ unreserved_keyword:
 			| DOUBLE_P
 			| DROP
 			| EACH
+			| ELEMENT
 			| ENABLE_P
 			| ENCODING
 			| ENCRYPTED
@@ -15857,6 +15901,7 @@ bare_label_keyword:
 			| DOUBLE_P
 			| DROP
 			| EACH
+			| ELEMENT
 			| ELSE
 			| ENABLE_P
 			| ENCODING
@@ -16876,6 +16921,27 @@ SplitColQualList(List *qualList,
 	*constraintList = qualList;
 }
 
+/* Split a list of FKColElem structs into separate name and reftype lists */
+static void
+SplitFKColElems(List *fkcolelems, List **names, List **reftypes)
+{
+	ListCell   *lc;
+
+	*names = NIL;
+	*reftypes = NIL;
+
+	Assert(**reftypes != NULL);
+	Assert(**names != NULL);
+
+	foreach(lc, fkcolelems)
+	{
+		FKColElem *fkcolelem = (FKColElem *) lfirst(lc);
+
+		*names = lappend(*names, fkcolelem->name);
+		*reftypes = lappend_int(*reftypes, fkcolelem->reftype);
+	}
+}
+
 /*
  * Process result of ConstraintAttributeSpec, and set appropriate bool flags
  * in the output command node.  Pass NULL for any flags the particular
```

<h3 id="parse_utilcmd-c"><code>src/backend/parser/parse_utilcmd.c</code></h3>

```diff
diff --git a/src/backend/parser/parse_utilcmd.c b/src/backend/parser/parse_utilcmd.c
index b31f3afa03..ccef8c8e0b 100644
--- a/src/backend/parser/parse_utilcmd.c
+++ b/src/backend/parser/parse_utilcmd.c
@@ -789,6 +789,8 @@ transformColumnDefinition(CreateStmtContext *cxt, ColumnDef *column)
 				 * list of FK constraints to be processed later.
 				 */
 				constraint->fk_attrs = list_make1(makeString(column->colname));
+				/* grammar should have set fk_reftypes */
+				Assert(list_length(constraint->fk_reftypes) == 1);
 				cxt->fkconstraints = lappend(cxt->fkconstraints, constraint);
 				break;
```

<h3 id="ri_triggers-c"><code>src/backend/utils/adt/ri_triggers.c</code></h3>

```diff
 diff --git a/src/backend/utils/adt/ri_triggers.c b/src/backend/utils/adt/ri_triggers.c
index 6e3a41062f..9e19df8403 100644
--- a/src/backend/utils/adt/ri_triggers.c
+++ b/src/backend/utils/adt/ri_triggers.c
@@ -108,9 +108,11 @@ typedef struct RI_ConstraintInfo
 	char		confupdtype;	/* foreign key's ON UPDATE action */
 	char		confdeltype;	/* foreign key's ON DELETE action */
 	char		confmatchtype;	/* foreign key's match type */
+	bool		has_array;		/* true if any reftype is EACH_ELEMENT */
 	int			nkeys;			/* number of key columns */
 	int16		pk_attnums[RI_MAX_NUMKEYS]; /* attnums of referenced cols */
 	int16		fk_attnums[RI_MAX_NUMKEYS]; /* attnums of referencing cols */
+	char		fk_reftypes[RI_MAX_NUMKEYS];	/* reference semantics */
 	Oid			pf_eq_oprs[RI_MAX_NUMKEYS]; /* equality operators (PK = FK) */
 	Oid			pp_eq_oprs[RI_MAX_NUMKEYS]; /* equality operators (PK = PK) */
 	Oid			ff_eq_oprs[RI_MAX_NUMKEYS]; /* equality operators (FK = FK) */
@@ -184,7 +186,8 @@ static void ri_GenerateQual(StringInfo buf,
 							const char *sep,
 							const char *leftop, Oid leftoptype,
 							Oid opoid,
-							const char *rightop, Oid rightoptype);
+							const char *rightop, Oid rightoptype,
+							char fkreftype);
 static void ri_GenerateQualCollation(StringInfo buf, Oid collation);
 static int	ri_NullCheck(TupleDesc tupdesc, TupleTableSlot *slot,
 						 const RI_ConstraintInfo *riinfo, bool rel_is_pk);
@@ -342,6 +345,7 @@ RI_FKey_check(TriggerData *trigdata)
 	if ((qplan = ri_FetchPreparedPlan(&qkey)) == NULL)
 	{
 		StringInfoData querybuf;
+		StringInfoData countbuf;
 		char		pkrelname[MAX_QUOTED_REL_NAME_LEN];
 		char		attname[MAX_QUOTED_NAME_LEN];
 		char		paramname[16];
@@ -355,6 +359,14 @@ RI_FKey_check(TriggerData *trigdata)
 		 *		   FOR KEY SHARE OF x
 		 * The type id's for the $ parameters are those of the
 		 * corresponding FK attributes.
+		 *
+		 * In case of an array ELEMENT foreign key, the previous query is used
+		 * to count the number of matching rows and see if every combination
+		 * is actually referenced.
+		 * The wrapping query is
+		 *	SELECT 1 WHERE
+		 *	  (SELECT count(DISTINCT y) FROM unnest($1) y)
+		 *	  = (SELECT count(*) FROM (<QUERY>) z)
 		 * ----------
 		 */
 		initStringInfo(&querybuf);
@@ -364,6 +376,13 @@ RI_FKey_check(TriggerData *trigdata)
 		appendStringInfo(&querybuf, "SELECT 1 FROM %s%s x",
 						 pk_only, pkrelname);
 		querysep = "WHERE";
+
+		if (riinfo->has_array)
+		{
+			initStringInfo(&countbuf);
+			appendStringInfo(&countbuf, "SELECT 1 WHERE ");
+		}
+
 		for (int i = 0; i < riinfo->nkeys; i++)
 		{
 			Oid			pk_type = RIAttType(pk_rel, riinfo->pk_attnums[i]);
@@ -372,18 +391,44 @@ RI_FKey_check(TriggerData *trigdata)
 			quoteOneName(attname,
 						 RIAttName(pk_rel, riinfo->pk_attnums[i]));
 			sprintf(paramname, "$%d", i + 1);
+
+			/*
+			 * In case of an array ELEMENT foreign key, we check that each
+			 * distinct non-null value in the array is present in the PK
+			 * table.
+			 */
+			if (riinfo->fk_reftypes[i] == FKCONSTR_REF_EACH_ELEMENT)
+				appendStringInfo(&countbuf,
+								 "(SELECT pg_catalog.count(DISTINCT y) FROM pg_catalog.unnest(%s) y)",
+								 paramname);
+
 			ri_GenerateQual(&querybuf, querysep,
 							attname, pk_type,
 							riinfo->pf_eq_oprs[i],
-							paramname, fk_type);
+							paramname, fk_type,
+							riinfo->fk_reftypes[i]);
 			querysep = "AND";
 			queryoids[i] = fk_type;
 		}
 		appendStringInfoString(&querybuf, " FOR KEY SHARE OF x");
 
-		/* Prepare and save the plan */
-		qplan = ri_PlanCheck(querybuf.data, riinfo->nkeys, queryoids,
-							 &qkey, fk_rel, pk_rel);
+		if (riinfo->has_array)
+		{
+			appendStringInfo(&countbuf,
+							 " OPERATOR(pg_catalog.=) (SELECT pg_catalog.count(*) FROM (%s) z)",
+							 querybuf.data);
+
+			/* Prepare and save the plan for Array Element Foreign Keys */
+			qplan = ri_PlanCheck(countbuf.data, riinfo->nkeys, queryoids,
+								 &qkey, fk_rel, pk_rel);
+		}
+		else
+		{
+			/* Prepare and save the plan */
+			qplan = ri_PlanCheck(querybuf.data, riinfo->nkeys, queryoids,
+								&qkey, fk_rel, pk_rel);
+
+		}
 	}
 
 	/*
@@ -502,7 +547,8 @@ ri_Check_Pk_Match(Relation pk_rel, Relation fk_rel,
 			ri_GenerateQual(&querybuf, querysep,
 							attname, pk_type,
 							riinfo->pp_eq_oprs[i],
-							paramname, pk_type);
+							paramname, pk_type,
+							FKCONSTR_REF_PLAIN);
 			querysep = "AND";
 			queryoids[i] = pk_type;
 		}
@@ -692,7 +738,8 @@ ri_restrict(TriggerData *trigdata, bool is_no_action)
 			ri_GenerateQual(&querybuf, querysep,
 							paramname, pk_type,
 							riinfo->pf_eq_oprs[i],
-							attname, fk_type);
+							attname, fk_type,
+							riinfo->fk_reftypes[i]);
 			if (pk_coll != fk_coll && !get_collation_isdeterministic(pk_coll))
 				ri_GenerateQualCollation(&querybuf, pk_coll);
 			querysep = "AND";
@@ -798,7 +845,8 @@ RI_FKey_cascade_del(PG_FUNCTION_ARGS)
 			ri_GenerateQual(&querybuf, querysep,
 							paramname, pk_type,
 							riinfo->pf_eq_oprs[i],
-							attname, fk_type);
+							attname, fk_type,
+							riinfo->fk_reftypes[i]);
 			if (pk_coll != fk_coll && !get_collation_isdeterministic(pk_coll))
 				ri_GenerateQualCollation(&querybuf, pk_coll);
 			querysep = "AND";
@@ -917,7 +965,8 @@ RI_FKey_cascade_upd(PG_FUNCTION_ARGS)
 			ri_GenerateQual(&qualbuf, qualsep,
 							paramname, pk_type,
 							riinfo->pf_eq_oprs[i],
-							attname, fk_type);
+							attname, fk_type,
+							riinfo->fk_reftypes[i]);
 			if (pk_coll != fk_coll && !get_collation_isdeterministic(pk_coll))
 				ri_GenerateQualCollation(&querybuf, pk_coll);
 			querysep = ",";
@@ -1097,7 +1146,8 @@ ri_set(TriggerData *trigdata, bool is_set_null)
 			ri_GenerateQual(&qualbuf, qualsep,
 							paramname, pk_type,
 							riinfo->pf_eq_oprs[i],
-							attname, fk_type);
+							attname, fk_type,
+							riinfo->fk_reftypes[i]);
 			if (pk_coll != fk_coll && !get_collation_isdeterministic(pk_coll))
 				ri_GenerateQualCollation(&querybuf, pk_coll);
 			querysep = ",";
@@ -1370,6 +1420,14 @@ RI_Initial_Check(Trigger *trigger, Relation fk_rel, Relation pk_rel)
 	 * For MATCH FULL:
 	 *	 (fk.keycol1 IS NOT NULL [OR ...])
 	 *
+	 * In case of an array ELEMENT column, relname is replaced with the
+	 * following subquery:
+	 *
+	 *	 SELECT unnest("keycol1") k1, "keycol1" ak1 [, ...]
+	 *	 FROM ONLY "public"."fk"
+	 *
+	 * where all the columns are renamed in order to prevent name collisions.
+	 *
 	 * We attach COLLATE clauses to the operators when comparing columns
 	 * that have different collations.
 	 *----------
@@ -1387,13 +1445,35 @@ RI_Initial_Check(Trigger *trigger, Relation fk_rel, Relation pk_rel)
 
 	quoteRelationName(pkrelname, pk_rel);
 	quoteRelationName(fkrelname, fk_rel);
-	fk_only = fk_rel->rd_rel->relkind == RELKIND_PARTITIONED_TABLE ?
-		"" : "ONLY ";
-	pk_only = pk_rel->rd_rel->relkind == RELKIND_PARTITIONED_TABLE ?
-		"" : "ONLY ";
-	appendStringInfo(&querybuf,
-					 " FROM %s%s fk LEFT OUTER JOIN %s%s pk ON",
-					 fk_only, fkrelname, pk_only, pkrelname);
+
+	if (riinfo->has_array)
+	{
+		appendStringInfo(&querybuf, " FROM ONLY %s fk", fkrelname);
+		for (int i = 0; i < riinfo->nkeys; i++)
+		{
+			if (riinfo->fk_reftypes[i] != FKCONSTR_REF_EACH_ELEMENT)
+				continue;
+
+			quoteOneName(fkattname,
+						 RIAttName(fk_rel, riinfo->fk_attnums[i]));
+			appendStringInfo(&querybuf,
+							 " CROSS JOIN LATERAL pg_catalog.unnest(fk.%s) a (e)",
+							 fkattname);
+		}
+		appendStringInfo(&querybuf,
+						 " LEFT OUTER JOIN ONLY %s pk ON",
+						 pkrelname);
+	}
+	else
+	{
+		fk_only = fk_rel->rd_rel->relkind == RELKIND_PARTITIONED_TABLE ?
+			"" : "ONLY ";
+		pk_only = pk_rel->rd_rel->relkind == RELKIND_PARTITIONED_TABLE ?
+			"" : "ONLY ";
+		appendStringInfo(&querybuf,
+						" FROM %s%s fk LEFT OUTER JOIN %s%s pk ON",
+						fk_only, fkrelname, pk_only, pkrelname);
+	}
 
 	strcpy(pkattname, "pk.");
 	strcpy(fkattname, "fk.");
@@ -1404,15 +1484,23 @@ RI_Initial_Check(Trigger *trigger, Relation fk_rel, Relation pk_rel)
 		Oid			fk_type = RIAttType(fk_rel, riinfo->fk_attnums[i]);
 		Oid			pk_coll = RIAttCollation(pk_rel, riinfo->pk_attnums[i]);
 		Oid			fk_coll = RIAttCollation(fk_rel, riinfo->fk_attnums[i]);
+		char	   *tmp;
 
 		quoteOneName(pkattname + 3,
 					 RIAttName(pk_rel, riinfo->pk_attnums[i]));
-		quoteOneName(fkattname + 3,
-					 RIAttName(fk_rel, riinfo->fk_attnums[i]));
+		if (riinfo->fk_reftypes[i] == FKCONSTR_REF_EACH_ELEMENT)
+			tmp = "a.e";
+		else
+		{
+			quoteOneName(fkattname + 3,
+						 RIAttName(fk_rel, riinfo->fk_attnums[i]));
+			tmp = fkattname;
+		}
 		ri_GenerateQual(&querybuf, sep,
 						pkattname, pk_type,
 						riinfo->pf_eq_oprs[i],
-						fkattname, fk_type);
+						tmp, fk_type,
+						FKCONSTR_REF_PLAIN);
 		if (pk_coll != fk_coll)
 			ri_GenerateQualCollation(&querybuf, pk_coll);
 		sep = "AND";
@@ -1428,10 +1516,15 @@ RI_Initial_Check(Trigger *trigger, Relation fk_rel, Relation pk_rel)
 	sep = "";
 	for (int i = 0; i < riinfo->nkeys; i++)
 	{
-		quoteOneName(fkattname, RIAttName(fk_rel, riinfo->fk_attnums[i]));
-		appendStringInfo(&querybuf,
-						 "%sfk.%s IS NOT NULL",
-						 sep, fkattname);
+		if (riinfo->fk_reftypes[i] == FKCONSTR_REF_EACH_ELEMENT)
+			appendStringInfo(&querybuf, "%sa.e IS NOT NULL", sep);
+		else
+		{
+			quoteOneName(fkattname, RIAttName(fk_rel, riinfo->fk_attnums[i]));
+			appendStringInfo(&querybuf,
+							 "%sfk.%s IS NOT NULL",
+							 sep, fkattname);
+		}
 		switch (riinfo->confmatchtype)
 		{
 			case FKCONSTR_MATCH_SIMPLE:
@@ -1647,7 +1740,8 @@ RI_PartitionRemove_Check(Trigger *trigger, Relation fk_rel, Relation pk_rel)
 		ri_GenerateQual(&querybuf, sep,
 						pkattname, pk_type,
 						riinfo->pf_eq_oprs[i],
-						fkattname, fk_type);
+						fkattname, fk_type,
+						riinfo->fk_reftypes[i]);
 		if (pk_coll != fk_coll)
 			ri_GenerateQualCollation(&querybuf, pk_coll);
 		sep = "AND";
@@ -1832,11 +1926,12 @@ ri_GenerateQual(StringInfo buf,
 				const char *sep,
 				const char *leftop, Oid leftoptype,
 				Oid opoid,
-				const char *rightop, Oid rightoptype)
+				const char *rightop, Oid rightoptype,
+				char fkreftype)
 {
 	appendStringInfo(buf, " %s ", sep);
 	generate_operator_clause(buf, leftop, leftoptype, opoid,
-							 rightop, rightoptype);
+							 rightop, rightoptype, fkreftype);
 }
 
 /*
@@ -2066,7 +2161,26 @@ ri_LoadConstraintInfo(Oid constraintOid)
 							   riinfo->pk_attnums,
 							   riinfo->pf_eq_oprs,
 							   riinfo->pp_eq_oprs,
-							   riinfo->ff_eq_oprs);
+							   riinfo->ff_eq_oprs,
+							   riinfo->fk_reftypes);
+
+	/*
+	 * Fix up some stuff for Array Element Foreign Keys.  We need a has_array
+	 * flag indicating whether there's an array foreign key, and we want to
+	 * set ff_eq_oprs[i] to array_eq() for array columns, because that's what
+	 * makes sense for ri_KeysEqual, and we have no other use for ff_eq_oprs
+	 * in this module.  (If we did, substituting the array comparator at the
+	 * call point in ri_KeysEqual might be more appropriate.)
+	 */
+	riinfo->has_array = false;
+	for (int i = 0; i < riinfo->nkeys; i++)
+	{
+		if (riinfo->fk_reftypes[i] == FKCONSTR_REF_EACH_ELEMENT)
+		{
+			riinfo->has_array = true;
+			riinfo->ff_eq_oprs[i] = ARRAY_EQ_OP;
+		}
+	}
 
 	ReleaseSysCache(tup);
```

<h3 id="ruleutils-c"><code>src/backend/utils/adt/ruleutils.c</code></h3>

```diff
diff --git a/src/backend/utils/adt/ruleutils.c b/src/backend/utils/adt/ruleutils.c
index 4a9244f4f6..f6a2a0c402 100644
--- a/src/backend/utils/adt/ruleutils.c
+++ b/src/backend/utils/adt/ruleutils.c
@@ -330,6 +330,9 @@ static char *pg_get_viewdef_worker(Oid viewoid,
 static char *pg_get_triggerdef_worker(Oid trigid, bool pretty);
 static int	decompile_column_index_array(Datum column_index_array, Oid relId,
 										 StringInfo buf);
+static void decompile_fk_column_index_array(Datum column_index_array,
+											Datum fk_reftype_array,
+											Oid relId, StringInfo buf);
 static char *pg_get_ruledef_worker(Oid ruleoid, int prettyFlags);
 static char *pg_get_indexdef_worker(Oid indexrelid, int colno,
 									const Oid *excludeOps,
@@ -2006,7 +2009,8 @@ pg_get_constraintdef_worker(Oid constraintId, bool fullCommand,
 	{
 		case CONSTRAINT_FOREIGN:
 			{
-				Datum		val;
+				Datum		colindexes;
+				Datum		reftypes;
 				bool		isnull;
 				const char *string;
 
@@ -2014,13 +2018,21 @@ pg_get_constraintdef_worker(Oid constraintId, bool fullCommand,
 				appendStringInfoString(&buf, "FOREIGN KEY (");
 
 				/* Fetch and build referencing-column list */
-				val = SysCacheGetAttr(CONSTROID, tup,
-									  Anum_pg_constraint_conkey, &isnull);
+				colindexes = SysCacheGetAttr(CONSTROID, tup,
+											 Anum_pg_constraint_conkey,
+											 &isnull);
 				if (isnull)
 					elog(ERROR, "null conkey for constraint %u",
 						 constraintId);
+				reftypes = SysCacheGetAttr(CONSTROID, tup,
+										   Anum_pg_constraint_confreftype,
+										   &isnull);
+				if (isnull)
+					elog(ERROR, "null confreftype for constraint %u",
+						 constraintId);
 
-				decompile_column_index_array(val, conForm->conrelid, &buf);
+				decompile_fk_column_index_array(colindexes, reftypes,
+												conForm->conrelid, &buf);
 
 				/* add foreign relation name */
 				appendStringInfo(&buf, ") REFERENCES %s(",
@@ -2028,13 +2040,15 @@ pg_get_constraintdef_worker(Oid constraintId, bool fullCommand,
 														NIL));
 
 				/* Fetch and build referenced-column list */
-				val = SysCacheGetAttr(CONSTROID, tup,
-									  Anum_pg_constraint_confkey, &isnull);
+				colindexes = SysCacheGetAttr(CONSTROID, tup,
+											 Anum_pg_constraint_confkey,
+											 &isnull);
 				if (isnull)
 					elog(ERROR, "null confkey for constraint %u",
 						 constraintId);
 
-				decompile_column_index_array(val, conForm->confrelid, &buf);
+				decompile_column_index_array(colindexes,
+											 conForm->confrelid, &buf);
 
 				appendStringInfoChar(&buf, ')');
 
@@ -2363,6 +2377,66 @@ decompile_column_index_array(Datum column_index_array, Oid relId,
 	return nKeys;
 }
 
+/*
+ * Convert an int16[] Datum and a char[] Datum into a comma-separated list of
+ * column names for the indicated relation, prefixed by appropriate keywords
+ * depending on the foreign key reference semantics indicated by the char[]
+ * entries.  Append the text to buf.
+ */
+static void
+decompile_fk_column_index_array(Datum column_index_array,
+								Datum fk_reftype_array,
+								Oid relId, StringInfo buf)
+{
+	Datum	   *keys;
+	int			nKeys;
+	Datum	   *reftypes;
+	int			nReftypes;
+	int			j;
+
+	/* Extract data from array of int16 */
+	deconstruct_array(DatumGetArrayTypeP(column_index_array),
+					  INT2OID, sizeof(int16), true, 's',
+					  &keys, NULL, &nKeys);
+
+	/* Extract data from array of char */
+	deconstruct_array(DatumGetArrayTypeP(fk_reftype_array),
+					  CHAROID, sizeof(char), true, 'c',
+					  &reftypes, NULL, &nReftypes);
+
+	if (nKeys != nReftypes)
+		elog(ERROR, "wrong confreftype cardinality");
+
+	for (j = 0; j < nKeys; j++)
+	{
+		char	   *colName;
+		const char *prefix;
+
+		colName = get_attname(relId, DatumGetInt16(keys[j]), false);
+
+		switch (DatumGetChar(reftypes[j]))
+		{
+			case FKCONSTR_REF_PLAIN:
+				prefix = "";
+				break;
+			case FKCONSTR_REF_EACH_ELEMENT:
+				prefix = "EACH ELEMENT OF ";
+				break;
+			default:
+				elog(ERROR, "invalid fk_reftype: %d",
+					 (int) DatumGetChar(reftypes[j]));
+				prefix = NULL;	/* keep compiler quiet */
+				break;
+		}
+
+		if (j == 0)
+			appendStringInfo(buf, "%s%s", prefix,
+							 quote_identifier(colName));
+		else
+			appendStringInfo(buf, ", %s%s", prefix,
+							 quote_identifier(colName));
+	}
+}
 
 /* ----------
  * pg_get_expr			- Decompile an expression tree
@@ -11391,16 +11465,24 @@ void
 generate_operator_clause(StringInfo buf,
 						 const char *leftop, Oid leftoptype,
 						 Oid opoid,
-						 const char *rightop, Oid rightoptype)
+						 const char *rightop, Oid rightoptype,
+						 char fkreftype)
 {
 	HeapTuple	opertup;
 	Form_pg_operator operform;
 	char	   *oprname;
 	char	   *nspname;
+	Oid			oproid;
+
+	/* Override operator with <<@ in case of FK array */
+	if(fkreftype == FKCONSTR_REF_EACH_ELEMENT)
+		oproid = OID_ARRAY_ELEMCONTAINED_OP;
+	else
+		oproid = opoid;
 
-	opertup = SearchSysCache1(OPEROID, ObjectIdGetDatum(opoid));
+	opertup = SearchSysCache1(OPEROID, ObjectIdGetDatum(oproid));
 	if (!HeapTupleIsValid(opertup))
-		elog(ERROR, "cache lookup failed for operator %u", opoid);
+		elog(ERROR, "cache lookup failed for operator %u", oproid);
 	operform = (Form_pg_operator) GETSTRUCT(opertup);
 	Assert(operform->oprkind == 'b');
 	oprname = NameStr(operform->oprname);
```

<h3 id="relcache-c"><code>src/backend/utils/cache/relcache.c</code></h3>

```diff
diff --git a/src/backend/utils/cache/relcache.c b/src/backend/utils/cache/relcache.c
index 7ef510cd01..b43ca0e927 100644
--- a/src/backend/utils/cache/relcache.c
+++ b/src/backend/utils/cache/relcache.c
@@ -4469,7 +4469,7 @@ RelationGetFKeyList(Relation relation)
 								   info->conkey,
 								   info->confkey,
 								   info->conpfeqop,
-								   NULL, NULL);
+								   NULL, NULL, NULL);
 
 		/* Add FK's node to the result list */
 		result = lappend(result, info);
```

<h3 id="pg_constraint-h"><code>src/include/catalog/pg_constraint.h</code></h3>

```diff
diff --git a/src/include/catalog/pg_constraint.h b/src/include/catalog/pg_constraint.h
index 63f0f8bf41..dd83ca6213 100644
--- a/src/include/catalog/pg_constraint.h
+++ b/src/include/catalog/pg_constraint.h
@@ -120,9 +120,19 @@ CATALOG(pg_constraint,2606,ConstraintRelationId)
 	 */
 	int16		confkey[1];
 
+	/*
+	 * If a foreign key, the reference semantics for each column.
+	 * 'p' to signify plain foreign key
+	 * 'e' to signify each element foreign key array
+	 */
+	char		confreftype[1];
+
 	/*
 	 * If a foreign key, the OIDs of the PK = FK equality operators for each
 	 * column of the constraint
+	 *
+	 * Note: for FArray Element Foreign Keys, all these operators are for the
+	 * array's element type.
 	 */
 	Oid			conpfeqop[1] BKI_LOOKUP(pg_operator);
 
@@ -219,6 +229,7 @@ extern Oid	CreateConstraintEntry(const char *constraintName,
 								  Oid indexRelId,
 								  Oid foreignRelId,
 								  const int16 *foreignKey,
+								  const char *foreignRefType,
 								  const Oid *pfEqOp,
 								  const Oid *ppEqOp,
 								  const Oid *ffEqOp,
@@ -259,7 +270,8 @@ extern Bitmapset *get_primary_key_attnos(Oid relid, bool deferrableOk,
 										 Oid *constraintOid);
 extern void DeconstructFkConstraintRow(HeapTuple tuple, int *numfks,
 									   AttrNumber *conkey, AttrNumber *confkey,
-									   Oid *pf_eq_oprs, Oid *pp_eq_oprs, Oid *ff_eq_oprs);
+									   Oid *pf_eq_oprs, Oid *pp_eq_oprs, Oid *ff_eq_oprs,
+									   char *fk_reftypes);
 
 extern bool check_functional_grouping(Oid relid,
 									  Index varno, Index varlevelsup,
```

<h3 id="parsenodes-h"><code>src/include/nodes/parsenodes.h</code></h3>

```diff
diff --git a/src/include/nodes/parsenodes.h b/src/include/nodes/parsenodes.h
index 236832a2ca..8fd49aacd4 100644
--- a/src/include/nodes/parsenodes.h
+++ b/src/include/nodes/parsenodes.h
@@ -2189,6 +2189,10 @@ typedef enum ConstrType			/* types of constraints */
 #define FKCONSTR_MATCH_PARTIAL		'p'
 #define FKCONSTR_MATCH_SIMPLE		's'
 
+ /* Foreign key column reference semantics codes */
+#define FKCONSTR_REF_PLAIN			'p'
+#define FKCONSTR_REF_EACH_ELEMENT	'e'
+
 typedef struct Constraint
 {
 	NodeTag		type;
@@ -2229,6 +2233,7 @@ typedef struct Constraint
 	RangeVar   *pktable;		/* Primary key table */
 	List	   *fk_attrs;		/* Attributes of foreign key */
 	List	   *pk_attrs;		/* Corresponding attrs in PK table */
+	List	   *fk_reftypes;	/* Per-column reference semantics (int List) */
 	char		fk_matchtype;	/* FULL, PARTIAL, SIMPLE */
 	char		fk_upd_action;	/* ON UPDATE action */
 	char		fk_del_action;	/* ON DELETE action */
```

<h3 id="kwlist-h"><code>src/include/parser/kwlist.h</code></h3>

```diff
diff --git a/src/include/parser/kwlist.h b/src/include/parser/kwlist.h
index 28083aaac9..58b937ddfa 100644
--- a/src/include/parser/kwlist.h
+++ b/src/include/parser/kwlist.h
@@ -142,6 +142,7 @@ PG_KEYWORD("domain", DOMAIN_P, UNRESERVED_KEYWORD, BARE_LABEL)
 PG_KEYWORD("double", DOUBLE_P, UNRESERVED_KEYWORD, BARE_LABEL)
 PG_KEYWORD("drop", DROP, UNRESERVED_KEYWORD, BARE_LABEL)
 PG_KEYWORD("each", EACH, UNRESERVED_KEYWORD, BARE_LABEL)
+PG_KEYWORD("element", ELEMENT, UNRESERVED_KEYWORD, BARE_LABEL)
 PG_KEYWORD("else", ELSE, RESERVED_KEYWORD, BARE_LABEL)
 PG_KEYWORD("enable", ENABLE_P, UNRESERVED_KEYWORD, BARE_LABEL)
 PG_KEYWORD("encoding", ENCODING, UNRESERVED_KEYWORD, BARE_LABEL)
```

<h3 id="builtins-h"><code>src/include/utils/builtins.h</code></h3>

```diff
diff --git a/src/include/utils/builtins.h b/src/include/utils/builtins.h
index 27d2f2ffb3..36d36d57b7 100644
--- a/src/include/utils/builtins.h
+++ b/src/include/utils/builtins.h
@@ -68,7 +68,8 @@ extern char *quote_qualified_identifier(const char *qualifier,
 extern void generate_operator_clause(fmStringInfo buf,
 									 const char *leftop, Oid leftoptype,
 									 Oid opoid,
-									 const char *rightop, Oid rightoptype);
+									 const char *rightop, Oid rightoptype,
+									 char fkreftype);
 
 /* varchar.c */
 extern int	bpchartruelen(char *s, int len);
```

<h3 id="element_fk-out"><code>src/test/regress/expected/element_fk.out</code></h3>

```diff
diff --git a/src/test/regress/expected/element_fk.out b/src/test/regress/expected/element_fk.out
new file mode 100644
index 0000000000..8be43b37a4
--- /dev/null
+++ b/src/test/regress/expected/element_fk.out
@@ -0,0 +1,617 @@
+-- EACH-ELEMENT FK CONSTRAINTS
+CREATE TABLE PKTABLEFORARRAY ( ptest1 int PRIMARY KEY, ptest2 text );
+-- Insert test data into PKTABLEFORARRAY
+INSERT INTO PKTABLEFORARRAY VALUES (1, 'Test1');
+INSERT INTO PKTABLEFORARRAY VALUES (2, 'Test2');
+INSERT INTO PKTABLEFORARRAY VALUES (3, 'Test3');
+INSERT INTO PKTABLEFORARRAY VALUES (4, 'Test4');
+INSERT INTO PKTABLEFORARRAY VALUES (5, 'Test5');
+-- Check alter table
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], ftest2 int );
+ALTER TABLE FKTABLEFORARRAY ADD CONSTRAINT FKARRAY FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY;
+DROP TABLE FKTABLEFORARRAY;
+-- Check alter table with rows
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], ftest2 int );
+INSERT INTO FKTABLEFORARRAY VALUES ('{1}', 1);
+ALTER TABLE FKTABLEFORARRAY ADD CONSTRAINT FKARRAY FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY;
+DROP TABLE FKTABLEFORARRAY;
+-- Check alter table with failing rows
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], ftest2 int );
+INSERT INTO FKTABLEFORARRAY VALUES ('{10,1}', 2);
+ALTER TABLE FKTABLEFORARRAY ADD CONSTRAINT FKARRAY FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY;
+ERROR:  insert or update on table "fktableforarray" violates foreign key constraint "fkarray"
+DETAIL:  Key (ftest1)=({10,1}) is not present in table "pktableforarray".
+DROP TABLE FKTABLEFORARRAY;
+-- Check create table
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY, ftest2 int );
+CREATE TABLE FKTABLEFORARRAYMDIM ( ftest1 int[][], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY, ftest2 int );
+CREATE TABLE FKTABLEFORARRAYNOTNULL ( ftest1 int[] NOT NULL, FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY, ftest2 int );
+-- Insert successful rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY VALUES ('{1}', 3);
+INSERT INTO FKTABLEFORARRAY VALUES ('{2}', 4);
+INSERT INTO FKTABLEFORARRAY VALUES ('{1}', 5);
+INSERT INTO FKTABLEFORARRAY VALUES ('{3}', 6);
+INSERT INTO FKTABLEFORARRAY VALUES ('{1}', 7);
+INSERT INTO FKTABLEFORARRAY VALUES ('{4,5}', 8);
+INSERT INTO FKTABLEFORARRAY VALUES ('{4,4}', 9);
+INSERT INTO FKTABLEFORARRAY VALUES (NULL, 10);
+INSERT INTO FKTABLEFORARRAY VALUES ('{}', 11);
+INSERT INTO FKTABLEFORARRAY VALUES ('{1,NULL}', 12);
+INSERT INTO FKTABLEFORARRAY VALUES ('{NULL}', 13);
+INSERT INTO FKTABLEFORARRAYMDIM VALUES ('{{4,5},{1,2},{1,3}}', 14);
+INSERT INTO FKTABLEFORARRAYMDIM VALUES ('{{4,5},{NULL,2},{NULL,3}}', 15);
+-- Insert failed rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY VALUES ('{6}', 16);
+ERROR:  insert or update on table "fktableforarray" violates foreign key constraint "fktableforarray_ftest1_fkey"
+DETAIL:  Key (ftest1)=({6}) is not present in table "pktableforarray".
+INSERT INTO FKTABLEFORARRAY VALUES ('{4,6}', 17);
+ERROR:  insert or update on table "fktableforarray" violates foreign key constraint "fktableforarray_ftest1_fkey"
+DETAIL:  Key (ftest1)=({4,6}) is not present in table "pktableforarray".
+INSERT INTO FKTABLEFORARRAY VALUES ('{6,NULL}', 18);
+ERROR:  insert or update on table "fktableforarray" violates foreign key constraint "fktableforarray_ftest1_fkey"
+DETAIL:  Key (ftest1)=({6,NULL}) is not present in table "pktableforarray".
+INSERT INTO FKTABLEFORARRAY VALUES ('{6,NULL,4,NULL}', 19);
+ERROR:  insert or update on table "fktableforarray" violates foreign key constraint "fktableforarray_ftest1_fkey"
+DETAIL:  Key (ftest1)=({6,NULL,4,NULL}) is not present in table "pktableforarray".
+INSERT INTO FKTABLEFORARRAYMDIM VALUES ('{{1,2},{6,NULL}}', 20);
+ERROR:  insert or update on table "fktableforarraymdim" violates foreign key constraint "fktableforarraymdim_ftest1_fkey"
+DETAIL:  Key (ftest1)=({{1,2},{6,NULL}}) is not present in table "pktableforarray".
+INSERT INTO FKTABLEFORARRAYNOTNULL VALUES (NULL, 21);
+ERROR:  null value in column "ftest1" of relation "fktableforarraynotnull" violates not-null constraint
+DETAIL:  Failing row contains (null, 21).
+-- Check FKTABLE
+SELECT * FROM FKTABLEFORARRAY;
+  ftest1  | ftest2 
+----------+--------
+ {1}      |      3
+ {2}      |      4
+ {1}      |      5
+ {3}      |      6
+ {1}      |      7
+ {4,5}    |      8
+ {4,4}    |      9
+          |     10
+ {}       |     11
+ {1,NULL} |     12
+ {NULL}   |     13
+(11 rows)
+
+-- Delete a row from PK TABLE (must fail due to ON DELETE NO ACTION)
+DELETE FROM PKTABLEFORARRAY WHERE ptest1=1;
+ERROR:  update or delete on table "pktableforarray" violates foreign key constraint "fktableforarray_ftest1_fkey" on table "fktableforarray"
+DETAIL:  Key (ptest1)=(1) is still referenced from table "fktableforarray".
+-- Check FKTABLE for removal of matched row
+SELECT * FROM FKTABLEFORARRAY;
+  ftest1  | ftest2 
+----------+--------
+ {1}      |      3
+ {2}      |      4
+ {1}      |      5
+ {3}      |      6
+ {1}      |      7
+ {4,5}    |      8
+ {4,4}    |      9
+          |     10
+ {}       |     11
+ {1,NULL} |     12
+ {NULL}   |     13
+(11 rows)
+
+-- Update a row from PK TABLE (must fail due to ON UPDATE NO ACTION)
+UPDATE PKTABLEFORARRAY SET ptest1=7 WHERE ptest1=1;
+ERROR:  update or delete on table "pktableforarray" violates foreign key constraint "fktableforarray_ftest1_fkey" on table "fktableforarray"
+DETAIL:  Key (ptest1)=(1) is still referenced from table "fktableforarray".
+-- Check FKTABLE for update of matched row
+SELECT * FROM FKTABLEFORARRAY;
+  ftest1  | ftest2 
+----------+--------
+ {1}      |      3
+ {2}      |      4
+ {1}      |      5
+ {3}      |      6
+ {1}      |      7
+ {4,5}    |      8
+ {4,4}    |      9
+          |     10
+ {}       |     11
+ {1,NULL} |     12
+ {NULL}   |     13
+(11 rows)
+
+-- Check UPDATE on FKTABLE
+UPDATE FKTABLEFORARRAY SET ftest1=ARRAY[1] WHERE ftest2=4;
+-- Check FKTABLE for update of matched row
+SELECT * FROM FKTABLEFORARRAY;
+  ftest1  | ftest2 
+----------+--------
+ {1}      |      3
+ {1}      |      5
+ {3}      |      6
+ {1}      |      7
+ {4,5}    |      8
+ {4,4}    |      9
+          |     10
+ {}       |     11
+ {1,NULL} |     12
+ {NULL}   |     13
+ {1}      |      4
+(11 rows)
+
+DROP TABLE FKTABLEFORARRAY;
+DROP TABLE FKTABLEFORARRAYNOTNULL;
+DROP TABLE FKTABLEFORARRAYMDIM;
+-- Allowed references with actions (NO ACTION, RESTRICT)
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE NO ACTION ON UPDATE NO ACTION, ftest2 int );
+DROP TABLE FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE NO ACTION ON UPDATE RESTRICT, ftest2 int );
+DROP TABLE FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE RESTRICT ON UPDATE NO ACTION, ftest2 int );
+DROP TABLE FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE RESTRICT ON UPDATE RESTRICT, ftest2 int );
+DROP TABLE FKTABLEFORARRAY;
+-- Not allowed references (SET NULL, SET DEFAULT, CASCADE)
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE NO ACTION ON UPDATE SET DEFAULT, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+NOTICE:  table "fktableforarray" does not exist, skipping
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE NO ACTION ON UPDATE SET NULL, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+NOTICE:  table "fktableforarray" does not exist, skipping
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE RESTRICT ON UPDATE SET DEFAULT, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+NOTICE:  table "fktableforarray" does not exist, skipping
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE RESTRICT ON UPDATE SET NULL, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+NOTICE:  table "fktableforarray" does not exist, skipping
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE CASCADE ON UPDATE NO ACTION, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE FKTABLEFORARRAY;
+ERROR:  table "fktableforarray" does not exist
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE CASCADE ON UPDATE RESTRICT, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE FKTABLEFORARRAY;
+ERROR:  table "fktableforarray" does not exist
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE CASCADE ON UPDATE SET DEFAULT, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+NOTICE:  table "fktableforarray" does not exist, skipping
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE CASCADE ON UPDATE SET NULL, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+NOTICE:  table "fktableforarray" does not exist, skipping
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET DEFAULT ON UPDATE NO ACTION, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+NOTICE:  table "fktableforarray" does not exist, skipping
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET DEFAULT ON UPDATE RESTRICT, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+NOTICE:  table "fktableforarray" does not exist, skipping
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET DEFAULT ON UPDATE SET DEFAULT, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+NOTICE:  table "fktableforarray" does not exist, skipping
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET DEFAULT ON UPDATE SET NULL, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+NOTICE:  table "fktableforarray" does not exist, skipping
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET NULL ON UPDATE NO ACTION, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+NOTICE:  table "fktableforarray" does not exist, skipping
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET NULL ON UPDATE RESTRICT, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+NOTICE:  table "fktableforarray" does not exist, skipping
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET NULL ON UPDATE SET DEFAULT, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+NOTICE:  table "fktableforarray" does not exist, skipping
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET NULL ON UPDATE SET NULL, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+NOTICE:  table "fktableforarray" does not exist, skipping
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE NO ACTION ON UPDATE CASCADE, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+NOTICE:  table "fktableforarray" does not exist, skipping
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE RESTRICT ON UPDATE CASCADE, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+NOTICE:  table "fktableforarray" does not exist, skipping
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE CASCADE ON UPDATE CASCADE, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+NOTICE:  table "fktableforarray" does not exist, skipping
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET DEFAULT ON UPDATE CASCADE, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+NOTICE:  table "fktableforarray" does not exist, skipping
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET NULL ON UPDATE CASCADE, ftest2 int );
+ERROR:  Array Element Foreign Keys support only NO ACTION and RESTRICT actions
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+NOTICE:  table "fktableforarray" does not exist, skipping
+-- Cleanup
+DROP TABLE PKTABLEFORARRAY;
+-- Check reference on empty table
+CREATE TABLE PKTABLEFORARRAY (ptest1 int PRIMARY KEY);
+CREATE TABLE FKTABLEFORARRAY  (ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY);
+INSERT INTO FKTABLEFORARRAY VALUES ('{}');
+DROP TABLE FKTABLEFORARRAY;
+DROP TABLE PKTABLEFORARRAY;
+-- Repeat a similar test using CHAR(1) keys rather than INTEGER
+CREATE TABLE PKTABLEFORARRAY ( ptest1 CHAR(1) PRIMARY KEY, ptest2 text );
+-- Populate the primary table
+INSERT INTO PKTABLEFORARRAY VALUES ('A', 'Test A');
+INSERT INTO PKTABLEFORARRAY VALUES ('B', 'Test B');
+INSERT INTO PKTABLEFORARRAY VALUES ('C', 'Test C');
+-- Create the refrencing table
+CREATE TABLE FKTABLEFORARRAY ( ftest1 char(1)[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON UPDATE RESTRICT ON DELETE RESTRICT, ftest2 int );
+-- Insert valid rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY VALUES ('{"A"}', 1);
+INSERT INTO FKTABLEFORARRAY VALUES ('{"B"}', 2);
+INSERT INTO FKTABLEFORARRAY VALUES ('{"C"}', 3);
+INSERT INTO FKTABLEFORARRAY VALUES ('{"A","B","C"}', 4);
+-- Insert invalid rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY VALUES ('{"D"}', 5);
+ERROR:  insert or update on table "fktableforarray" violates foreign key constraint "fktableforarray_ftest1_fkey"
+DETAIL:  Key (ftest1)=({D}) is not present in table "pktableforarray".
+INSERT INTO FKTABLEFORARRAY VALUES ('{"A","B","D"}', 6);
+ERROR:  insert or update on table "fktableforarray" violates foreign key constraint "fktableforarray_ftest1_fkey"
+DETAIL:  Key (ftest1)=({A,B,D}) is not present in table "pktableforarray".
+-- Check FKTABLE
+SELECT * FROM FKTABLEFORARRAY;
+ ftest1  | ftest2 
+---------+--------
+ {A}     |      1
+ {B}     |      2
+ {C}     |      3
+ {A,B,C} |      4
+(4 rows)
+
+-- Delete a row from PK TABLE (must fail due to ON DELETE RESTRICT)
+DELETE FROM PKTABLEFORARRAY WHERE ptest1='A';
+ERROR:  update or delete on table "pktableforarray" violates foreign key constraint "fktableforarray_ftest1_fkey" on table "fktableforarray"
+DETAIL:  Key (ptest1)=(A) is still referenced from table "fktableforarray".
+-- Check FKTABLE for removal of matched row
+SELECT * FROM FKTABLEFORARRAY;
+ ftest1  | ftest2 
+---------+--------
+ {A}     |      1
+ {B}     |      2
+ {C}     |      3
+ {A,B,C} |      4
+(4 rows)
+
+-- Update a row from PK TABLE (must fail due to ON UPDATE RESTRICT)
+UPDATE PKTABLEFORARRAY SET ptest1='D' WHERE ptest1='B';
+ERROR:  update or delete on table "pktableforarray" violates foreign key constraint "fktableforarray_ftest1_fkey" on table "fktableforarray"
+DETAIL:  Key (ptest1)=(B) is still referenced from table "fktableforarray".
+-- Check FKTABLE for update of matched row
+SELECT * FROM FKTABLEFORARRAY;
+ ftest1  | ftest2 
+---------+--------
+ {A}     |      1
+ {B}     |      2
+ {C}     |      3
+ {A,B,C} |      4
+(4 rows)
+
+-- Cleanup
+DROP TABLE FKTABLEFORARRAY;
+DROP TABLE PKTABLEFORARRAY;
+-- Composite primary keys
+CREATE TABLE PKTABLEFORARRAY ( id1 CHAR(1), id2 CHAR(1), ptest2 text, PRIMARY KEY (id1, id2) );
+-- Populate the primary table
+INSERT INTO PKTABLEFORARRAY VALUES ('A', 'A', 'Test A');
+INSERT INTO PKTABLEFORARRAY VALUES ('A', 'B', 'Test B');
+INSERT INTO PKTABLEFORARRAY VALUES ('B', 'C', 'Test B');
+-- Create the refrencing table
+CREATE TABLE FKTABLEFORARRAY ( fid1 CHAR(1), fid2 CHAR(1)[], ftest2 text, FOREIGN KEY (fid1, EACH ELEMENT OF fid2) REFERENCES PKTABLEFORARRAY);
+-- Insert valid rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY VALUES ('A', ARRAY['A','B'], '1');
+INSERT INTO FKTABLEFORARRAY VALUES ('B', ARRAY['C'], '2');
+-- Insert invalid rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY VALUES ('A', ARRAY['A','B', 'C'], '3');
+ERROR:  insert or update on table "fktableforarray" violates foreign key constraint "fktableforarray_fid1_fid2_fkey"
+DETAIL:  Key (fid1, fid2)=(A, {A,B,C}) is not present in table "pktableforarray".
+INSERT INTO FKTABLEFORARRAY VALUES ('B', ARRAY['A'], '4');
+ERROR:  insert or update on table "fktableforarray" violates foreign key constraint "fktableforarray_fid1_fid2_fkey"
+DETAIL:  Key (fid1, fid2)=(B, {A}) is not present in table "pktableforarray".
+-- Cleanup
+DROP TABLE FKTABLEFORARRAY;
+DROP TABLE PKTABLEFORARRAY;
+-- Test Array Element Foreign Keys with composite type
+CREATE TYPE INVOICEID AS (year_part INTEGER, progressive_part INTEGER);
+CREATE TABLE PKTABLEFORARRAY ( id INVOICEID PRIMARY KEY,  ptest2 text);
+-- Populate the primary table
+INSERT INTO PKTABLEFORARRAY VALUES (ROW(2010, 99), 'Last invoice for 2010');
+INSERT INTO PKTABLEFORARRAY VALUES (ROW(2011, 1), 'First invoice for 2011');
+INSERT INTO PKTABLEFORARRAY VALUES (ROW(2011, 2), 'Second invoice for 2011');
+-- Create the refrencing table
+CREATE TABLE FKTABLEFORARRAY ( id SERIAL PRIMARY KEY, invoice_ids INVOICEID[], FOREIGN KEY (EACH ELEMENT OF invoice_ids) REFERENCES PKTABLEFORARRAY, ftest2 TEXT );
+-- Insert valid rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY(invoice_ids, ftest2) VALUES (ARRAY['(2010,99)']::INVOICEID[], 'Product A');
+INSERT INTO FKTABLEFORARRAY(invoice_ids, ftest2) VALUES (ARRAY['(2011,1)','(2011,2)']::INVOICEID[], 'Product B');
+INSERT INTO FKTABLEFORARRAY(invoice_ids, ftest2) VALUES (ARRAY['(2011,2)']::INVOICEID[], 'Product C');
+-- Insert invalid rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY(invoice_ids, ftest2) VALUES (ARRAY['(2011,99)']::INVOICEID[], 'Product A');
+ERROR:  insert or update on table "fktableforarray" violates foreign key constraint "fktableforarray_invoice_ids_fkey"
+DETAIL:  Key (invoice_ids)=({"(2011,99)"}) is not present in table "pktableforarray".
+INSERT INTO FKTABLEFORARRAY(invoice_ids, ftest2) VALUES (ARRAY['(2011,1)','(2010,1)']::INVOICEID[], 'Product B');
+ERROR:  insert or update on table "fktableforarray" violates foreign key constraint "fktableforarray_invoice_ids_fkey"
+DETAIL:  Key (invoice_ids)=({"(2011,1)","(2010,1)"}) is not present in table "pktableforarray".
+-- Check FKTABLE
+SELECT * FROM FKTABLEFORARRAY;
+ id |       invoice_ids       |  ftest2   
+----+-------------------------+-----------
+  1 | {"(2010,99)"}           | Product A
+  2 | {"(2011,1)","(2011,2)"} | Product B
+  3 | {"(2011,2)"}            | Product C
+(3 rows)
+
+-- Delete a row from PK TABLE
+DELETE FROM PKTABLEFORARRAY WHERE id=ROW(2010,99);
+ERROR:  update or delete on table "pktableforarray" violates foreign key constraint "fktableforarray_invoice_ids_fkey" on table "fktableforarray"
+DETAIL:  Key (id)=((2010,99)) is still referenced from table "fktableforarray".
+-- Check FKTABLE for removal of matched row
+SELECT * FROM FKTABLEFORARRAY;
+ id |       invoice_ids       |  ftest2   
+----+-------------------------+-----------
+  1 | {"(2010,99)"}           | Product A
+  2 | {"(2011,1)","(2011,2)"} | Product B
+  3 | {"(2011,2)"}            | Product C
+(3 rows)
+
+-- Update a row from PK TABLE
+UPDATE PKTABLEFORARRAY SET id=ROW(2011,99) WHERE id=ROW(2011,1);
+ERROR:  update or delete on table "pktableforarray" violates foreign key constraint "fktableforarray_invoice_ids_fkey" on table "fktableforarray"
+DETAIL:  Key (id)=((2011,1)) is still referenced from table "fktableforarray".
+-- Check FKTABLE for update of matched row
+SELECT * FROM FKTABLEFORARRAY;
+ id |       invoice_ids       |  ftest2   
+----+-------------------------+-----------
+  1 | {"(2010,99)"}           | Product A
+  2 | {"(2011,1)","(2011,2)"} | Product B
+  3 | {"(2011,2)"}            | Product C
+(3 rows)
+
+-- Cleanup
+DROP TABLE FKTABLEFORARRAY;
+DROP TABLE PKTABLEFORARRAY;
+DROP TYPE INVOICEID;
+-- Check for an array column referencing another array column (NOT ELEMENT FOREIGN KEY)
+-- Create primary table with an array primary key
+CREATE TABLE PKTABLEFORARRAY ( id INT[] PRIMARY KEY,  ptest2 text);
+-- Create the refrencing table
+CREATE TABLE FKTABLEFORARRAY ( id SERIAL PRIMARY KEY, fids INT[] REFERENCES PKTABLEFORARRAY, ftest2 TEXT );
+-- Populate the primary table
+INSERT INTO PKTABLEFORARRAY VALUES ('{1,1}', 'A');
+INSERT INTO PKTABLEFORARRAY VALUES ('{1,2}', 'B');
+-- Insert valid rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY (fids, ftest2) VALUES ('{1,1}', 'Product A');
+INSERT INTO FKTABLEFORARRAY (fids, ftest2) VALUES ('{1,2}', 'Product B');
+-- Insert invalid rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY (fids, ftest2) VALUES ('{0,1}', 'Product C');
+ERROR:  insert or update on table "fktableforarray" violates foreign key constraint "fktableforarray_fids_fkey"
+DETAIL:  Key (fids)=({0,1}) is not present in table "pktableforarray".
+INSERT INTO FKTABLEFORARRAY (fids, ftest2) VALUES ('{2,1}', 'Product D');
+ERROR:  insert or update on table "fktableforarray" violates foreign key constraint "fktableforarray_fids_fkey"
+DETAIL:  Key (fids)=({2,1}) is not present in table "pktableforarray".
+-- Cleanup
+DROP TABLE FKTABLEFORARRAY;
+DROP TABLE PKTABLEFORARRAY;
+-- ---------------------------------------
+-- Multi-column "ELEMENT" foreign key tests
+-- ---------------------------------------
+-- Create DIM1 table with two-column primary key
+CREATE TABLE DIM1 (X INTEGER NOT NULL, Y INTEGER NOT NULL, PRIMARY KEY (X, Y));
+-- Populate DIM1 table pairs
+INSERT INTO DIM1 SELECT x.t, x.t * y.t
+	FROM (SELECT generate_series(1, 10) AS t) x,
+	(SELECT generate_series(0, 10) AS t) y;
+-- Test with TABLE declaration of an element foreign key constraint (NO ACTION)
+CREATE TABLE F1 (
+	x INTEGER PRIMARY KEY, y INTEGER[],
+	FOREIGN KEY (x, EACH ELEMENT OF y) REFERENCES DIM1(x, y)
+);
+-- Insert facts
+INSERT INTO F1 VALUES (1, '{0,1,2,3,4,5}'); -- OK
+INSERT INTO F1 VALUES (2, '{0,2,4,6}'); -- OK
+INSERT INTO F1 VALUES (3, '{0,3,6,9,0,3,6,9,0,0,0,0,9,9}'); -- OK (multiple occurrences)
+INSERT INTO F1 VALUES (4, '{0,2,4}'); -- FAILS (2 is not present)
+ERROR:  insert or update on table "f1" violates foreign key constraint "f1_x_y_fkey"
+DETAIL:  Key (x, y)=(4, {0,2,4}) is not present in table "dim1".
+INSERT INTO F1 VALUES (4, '{0,NULL,4}'); -- OK
+INSERT INTO F1 VALUES (5, '{0,NULL,5}'); -- OK
+-- Try updates
+UPDATE F1 SET y = '{0,2,4,6}' WHERE x = 2; -- OK
+UPDATE F1 SET y = '{0,2,3,4,6}' WHERE x = 2; -- FAILS
+ERROR:  insert or update on table "f1" violates foreign key constraint "f1_x_y_fkey"
+DETAIL:  Key (x, y)=(2, {0,2,3,4,6}) is not present in table "dim1".
+UPDATE F1 SET x = 20, y = '{0,2,3,4,6}' WHERE x = 2; -- FAILS (20 does not exist)
+ERROR:  insert or update on table "f1" violates foreign key constraint "f1_x_y_fkey"
+DETAIL:  Key (x, y)=(20, {0,2,3,4,6}) is not present in table "dim1".
+UPDATE F1 SET y = '{0,4,8}' WHERE x = 4; -- OK
+UPDATE F1 SET y = '{0,5,NULL,10}' WHERE x = 5; -- OK
+DROP TABLE F1;
+-- Test with FOREIGN KEY after TABLE population
+CREATE TABLE F1 (
+	x INTEGER PRIMARY KEY, y INTEGER[]
+);
+-- Insert facts
+INSERT INTO F1 VALUES (1, '{0,1,2,3,4,5}'); -- OK
+INSERT INTO F1 VALUES (2, '{0,2,4,6}'); -- OK
+INSERT INTO F1 VALUES (3, '{0,3,6,9,0,3,6,9,0,0,0,0,9,9}'); -- OK (multiple occurrences)
+INSERT INTO F1 VALUES (4, '{0,2,4}'); -- OK (2 is not present)
+INSERT INTO F1 VALUES (5, '{0,NULL,5}'); -- OK
+-- Add foreign key (FAILS)
+ALTER TABLE F1 ADD FOREIGN KEY (x, EACH ELEMENT OF y) REFERENCES DIM1(x, y);
+ERROR:  insert or update on table "f1" violates foreign key constraint "f1_x_y_fkey"
+DETAIL:  Key (x, y)=(4, {0,2,4}) is not present in table "dim1".
+DROP TABLE F1;
+-- Test with TABLE declaration of a two-dim ELEMENT foreign key constraint (FAILS)
+CREATE TABLE F1 (
+	x INTEGER[] PRIMARY KEY, y INTEGER[],
+	FOREIGN KEY (EACH ELEMENT OF x, EACH ELEMENT OF y) REFERENCES DIM1(x, y)
+);
+ERROR:  foreign keys support only one array column
+-- Test with two-dim ELEMENT foreign key after TABLE population
+CREATE TABLE F1 (
+	x INTEGER[] PRIMARY KEY, y INTEGER[]
+);
+INSERT INTO F1 VALUES ('{1}', '{0,1,2,3,4,5}'); -- OK
+INSERT INTO F1 VALUES ('{1,2}', '{0,2,4,6}'); -- OK
+-- Add foreign key (FAILS)
+ALTER TABLE F1 ADD FOREIGN KEY (EACH ELEMENT OF x, EACH ELEMENT OF y) REFERENCES DIM1(x, y);
+ERROR:  foreign keys support only one array column
+DROP TABLE F1;
+-- Cleanup
+DROP TABLE DIM1;
+-- Check for potential name conflicts (with internal integrity checks)
+CREATE TABLE x1(x1 int, x2 int, PRIMARY KEY(x1,x2));
+INSERT INTO x1 VALUES
+       (1,4),
+       (1,5),
+       (2,4),
+       (2,5),
+       (3,6),
+       (3,7)
+;
+CREATE TABLE x2(x1 int[], x2 int, FOREIGN KEY(EACH ELEMENT OF x1, x2) REFERENCES x1);
+INSERT INTO x2 VALUES ('{1,2}',4);
+INSERT INTO x2 VALUES ('{1,3}',6); -- FAILS
+ERROR:  insert or update on table "x2" violates foreign key constraint "x2_x1_x2_fkey"
+DETAIL:  Key (x1, x2)=({1,3}, 6) is not present in table "x1".
+DROP TABLE x2;
+CREATE TABLE x2(x1 int[], x2 int);
+INSERT INTO x2 VALUES ('{1,2}',4);
+INSERT INTO x2 VALUES ('{1,3}',6);
+ALTER TABLE x2 ADD CONSTRAINT fk_const FOREIGN KEY(EACH ELEMENT OF x1, x2) REFERENCES x1;  -- FAILS
+ERROR:  insert or update on table "x2" violates foreign key constraint "fk_const"
+DETAIL:  Key (x1, x2)=({1,3}, 6) is not present in table "x1".
+DROP TABLE x2;
+DROP TABLE x1;
+-- ---------------------------------------
+-- Multi-dimensional "ELEMENT" foreign key tests
+-- ---------------------------------------
+-- Create DIM1 table with two-column primary key
+CREATE TABLE DIM1 (X INTEGER NOT NULL PRIMARY KEY,
+	CODE TEXT NOT NULL UNIQUE);
+-- Populate DIM1 table pairs
+INSERT INTO DIM1 SELECT t, 'DIM1-' || lpad(t::TEXT, 2, '0')
+	FROM (SELECT generate_series(1, 10)) x(t);
+-- Test with TABLE declaration of an element foreign key constraint (NO ACTION)
+CREATE TABLE F1 (
+	ID SERIAL PRIMARY KEY,
+	SLOTS INTEGER[3][3], FOREIGN KEY (EACH ELEMENT OF SLOTS) REFERENCES DIM1
+);
+INSERT INTO F1(SLOTS) VALUES ('{{NULL, 1, NULL}, {NULL, NULL, 3}, {NULL, NULL, 6}}'); -- OK
+INSERT INTO F1(SLOTS) VALUES ('{{NULL, 1, NULL}, {NULL, NULL, 11}, {NULL, NULL, 6}}'); -- FAILS
+ERROR:  insert or update on table "f1" violates foreign key constraint "f1_slots_fkey"
+DETAIL:  Key (slots)=({{NULL,1,NULL},{NULL,NULL,11},{NULL,NULL,6}}) is not present in table "dim1".
+INSERT INTO F1(SLOTS) VALUES ('{{1, 2, 3}, {4, 5, 6}, {7, 8, 9}}'); -- OK
+INSERT INTO F1(SLOTS) VALUES ('{1, 2, 3, 4, 5, 6, 7, 8, 9}'); -- OK
+UPDATE F1 SET SLOTS = '{{NULL, 1, NULL}, {NULL, NULL, 3}, {7, 8, 10}}' WHERE ID = 1; -- OK
+UPDATE F1 SET SLOTS = '{{100, 100, 100}, {NULL, NULL, 20}, {7, 8, 10}}' WHERE ID = 1; -- FAILS
+ERROR:  insert or update on table "f1" violates foreign key constraint "f1_slots_fkey"
+DETAIL:  Key (slots)=({{100,100,100},{NULL,NULL,20},{7,8,10}}) is not present in table "dim1".
+DROP TABLE F1;
+-- Test with postponed foreign key
+CREATE TABLE F1 (
+	ID SERIAL PRIMARY KEY,
+	SLOTS INTEGER[3][3]
+);
+INSERT INTO F1(SLOTS) VALUES ('{{NULL, 1, NULL}, {NULL, NULL, 3}, {NULL, NULL, 6}}'); -- OK
+INSERT INTO F1(SLOTS) VALUES ('{{NULL, 1, NULL}, {NULL, NULL, 11}, {NULL, NULL, 6}}'); -- OK
+INSERT INTO F1(SLOTS) VALUES ('{{1, 2, 3}, {4, 5, 6}, {7, 8, 9}}'); -- OK
+INSERT INTO F1(SLOTS) VALUES ('{1, 2, 3, 4, 5, 6, 7, 8, 9}'); -- OK
+ALTER TABLE F1 ADD FOREIGN KEY (EACH ELEMENT OF SLOTS) REFERENCES DIM1; -- FAILS
+ERROR:  insert or update on table "f1" violates foreign key constraint "f1_slots_fkey"
+DETAIL:  Key (slots)=({{NULL,1,NULL},{NULL,NULL,11},{NULL,NULL,6}}) is not present in table "dim1".
+DELETE FROM F1 WHERE ID = 2; -- REMOVE ISSUE
+ALTER TABLE F1 ADD FOREIGN KEY (EACH ELEMENT OF SLOTS) REFERENCES DIM1; -- NOW OK
+INSERT INTO F1(SLOTS) VALUES ('{{NULL, 1, NULL}, {NULL, NULL, 11}, {NULL, NULL, 6}}'); -- FAILS
+ERROR:  insert or update on table "f1" violates foreign key constraint "f1_slots_fkey"
+DETAIL:  Key (slots)=({{NULL,1,NULL},{NULL,NULL,11},{NULL,NULL,6}}) is not present in table "dim1".
+DROP TABLE F1;
+-- Leave tables in the database
+CREATE TABLE PKTABLEFORELEMENTFK ( ptest1 int PRIMARY KEY, ptest2 text );
+CREATE TABLE FKTABLEFORELEMENTFK ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORELEMENTFK, ftest2 int );
+-- Check ALTER TABLE ALTER TYPE
+ALTER TABLE FKTABLEFORELEMENTFK ALTER FTEST1 TYPE INT[];
+-- Check GIN index
+-- Define PKTABLEFORARRAYGIN
+CREATE TABLE PKTABLEFORARRAYGIN ( ptest1 int PRIMARY KEY, ptest2 text );
+-- Insert test data into PKTABLEFORARRAYGIN
+INSERT INTO PKTABLEFORARRAYGIN VALUES (1, 'Test1');
+INSERT INTO PKTABLEFORARRAYGIN VALUES (2, 'Test2');
+INSERT INTO PKTABLEFORARRAYGIN VALUES (3, 'Test3');
+INSERT INTO PKTABLEFORARRAYGIN VALUES (4, 'Test4');
+INSERT INTO PKTABLEFORARRAYGIN VALUES (5, 'Test5');
+INSERT INTO PKTABLEFORARRAYGIN VALUES (6, 'Test6');
+-- Define FKTABLEFORARRAYGIN
+CREATE TABLE FKTABLEFORARRAYGIN ( ftest1 int[],
+    ftest2 int PRIMARY KEY,
+    FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAYGIN
+    ON DELETE NO ACTION ON UPDATE NO ACTION);
+-- -- Create index on FKTABLEFORARRAYGIN
+CREATE INDEX ON FKTABLEFORARRAYGIN USING gin (ftest1 array_ops);
+-- Populate Table
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{5}', 1);
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{3,2}', 2);
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{3,5,2,5}', 3);
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{3,4,4}', 4);
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{3,5,4,1,3}', 5);
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{1}', 6);
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{5,1}', 7);
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{2,1,2,4,1}', 8);
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{4,2}', 9);
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{3,4,5,3}', 10);
+-- Try UPDATE
+UPDATE PKTABLEFORARRAYGIN SET ptest1=7 WHERE ptest1=6;
+-- Try using the indexable operator
+SELECT * FROM FKTABLEFORARRAYGIN WHERE ftest1 @>> 5;
+   ftest1    | ftest2 
+-------------+--------
+ {5}         |      1
+ {3,5,2,5}   |      3
+ {3,5,4,1,3} |      5
+ {5,1}       |      7
+ {3,4,5,3}   |     10
+(5 rows)
+
+-- Cleanup
+DROP TABLE FKTABLEFORARRAYGIN;
+DROP TABLE PKTABLEFORARRAYGIN;
+-- ---------------------------------------
+-- Invalid refrencing key tests
+-- ---------------------------------------
+CREATE TABLE PKTABLEVIOLATING ( ptest1 int PRIMARY KEY, ptest2 text );
+-- Attempt fk constraint between int <-> int
+CREATE TABLE FKTABLEVIOLATING ( ftest1 int, FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORELEMENTFK, ftest2 int );
+ERROR:  foreign key constraint "fktableviolating_ftest1_fkey" cannot be implemented
+DETAIL:  Key column "ftest1" has type integer which is not an array type.
+-- Attempt fk constraint between int <-> char[]
+CREATE TABLE FKTABLEVIOLATING ( ftest1 char[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORELEMENTFK, ftest2 int );
+ERROR:  foreign key constraint "fktableviolating_ftest1_fkey" cannot be implemented
+DETAIL:  Unssupported relation between character[] and integer.
+-- Attempt fk constraint between int <-> char
+CREATE TABLE FKTABLEVIOLATING ( ftest1 char, FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORELEMENTFK, ftest2 int );
+ERROR:  foreign key constraint "fktableviolating_ftest1_fkey" cannot be implemented
+DETAIL:  Key column "ftest1" has type character which is not an array type.
+-- Attempt fk constraint between int <-> smallint[]
+CREATE TABLE FKTABLEVIOLATING ( ftest1 smallint[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORELEMENTFK, ftest2 int );
+ERROR:  foreign key constraint "fktableviolating_ftest1_fkey" cannot be implemented
+DETAIL:  Unssupported relation between smallint[] and integer.
+-- Attempt fk constraint between int <-> bigint[]
+CREATE TABLE FKTABLEVIOLATING ( ftest1 bigint[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORELEMENTFK, ftest2 int );
+ERROR:  foreign key constraint "fktableviolating_ftest1_fkey" cannot be implemented
+DETAIL:  Unssupported relation between bigint[] and integer.
+-- Attempt fk constraint between int <-> int2vector
+CREATE TABLE FKTABLEVIOLATING ( ftest1 int2vector, FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORELEMENTFK, ftest2 int );
+ERROR:  foreign key constraint "fktableviolating_ftest1_fkey" cannot be implemented
+DETAIL:  Unssupported relation between int2vector and integer.
```

<h3 id="parallel_schedule"><code>src/test/regress/parallel_schedule</code></h3>

```diff
diff --git a/src/test/regress/parallel_schedule b/src/test/regress/parallel_schedule
index 12bb67e491..62443a5ede 100644
--- a/src/test/regress/parallel_schedule
+++ b/src/test/regress/parallel_schedule
@@ -80,7 +80,7 @@ test: brin gin gist spgist privileges init_privs security_label collate matview
 # ----------
 # Another group of parallel tests
 # ----------
-test: create_table_like alter_generic alter_operator misc async dbsize misc_functions sysviews tsrf tid tidscan collate.icu.utf8 incremental_sort
+test: create_table_like alter_generic alter_operator misc async dbsize misc_functions sysviews tsrf tid tidscan collate.icu.utf8 incremental_sort element_fk
 
 # rules cannot run concurrently with any test that creates
 # a view or rule in the public schema
```

<h3 id="serial_schedule"><code>src/test/regress/serial_schedule</code></h3>

```diff
diff --git a/src/test/regress/serial_schedule b/src/test/regress/serial_schedule
index 59b416fd80..603cc92a51 100644
--- a/src/test/regress/serial_schedule
+++ b/src/test/regress/serial_schedule
@@ -139,6 +139,7 @@ test: tsrf
 test: tid
 test: tidscan
 test: collate.icu.utf8
+test: element_fk
 test: rules
 test: psql
 test: psql_crosstab
```

<h3 id="element_fk-sql"><code>src/test/regress/sql/element_fk.sql</code></h3>

```diff
diff --git a/src/test/regress/sql/element_fk.sql b/src/test/regress/sql/element_fk.sql
new file mode 100644
index 0000000000..62242f78b7
--- /dev/null
+++ b/src/test/regress/sql/element_fk.sql
@@ -0,0 +1,473 @@
+-- EACH-ELEMENT FK CONSTRAINTS
+
+CREATE TABLE PKTABLEFORARRAY ( ptest1 int PRIMARY KEY, ptest2 text );
+
+-- Insert test data into PKTABLEFORARRAY
+INSERT INTO PKTABLEFORARRAY VALUES (1, 'Test1');
+INSERT INTO PKTABLEFORARRAY VALUES (2, 'Test2');
+INSERT INTO PKTABLEFORARRAY VALUES (3, 'Test3');
+INSERT INTO PKTABLEFORARRAY VALUES (4, 'Test4');
+INSERT INTO PKTABLEFORARRAY VALUES (5, 'Test5');
+
+-- Check alter table
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], ftest2 int );
+ALTER TABLE FKTABLEFORARRAY ADD CONSTRAINT FKARRAY FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY;
+DROP TABLE FKTABLEFORARRAY;
+
+-- Check alter table with rows
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], ftest2 int );
+INSERT INTO FKTABLEFORARRAY VALUES ('{1}', 1);
+ALTER TABLE FKTABLEFORARRAY ADD CONSTRAINT FKARRAY FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY;
+DROP TABLE FKTABLEFORARRAY;
+
+-- Check alter table with failing rows
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], ftest2 int );
+INSERT INTO FKTABLEFORARRAY VALUES ('{10,1}', 2);
+ALTER TABLE FKTABLEFORARRAY ADD CONSTRAINT FKARRAY FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY;
+DROP TABLE FKTABLEFORARRAY;
+
+-- Check create table
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY, ftest2 int );
+CREATE TABLE FKTABLEFORARRAYMDIM ( ftest1 int[][], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY, ftest2 int );
+CREATE TABLE FKTABLEFORARRAYNOTNULL ( ftest1 int[] NOT NULL, FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY, ftest2 int );
+
+-- Insert successful rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY VALUES ('{1}', 3);
+INSERT INTO FKTABLEFORARRAY VALUES ('{2}', 4);
+INSERT INTO FKTABLEFORARRAY VALUES ('{1}', 5);
+INSERT INTO FKTABLEFORARRAY VALUES ('{3}', 6);
+INSERT INTO FKTABLEFORARRAY VALUES ('{1}', 7);
+INSERT INTO FKTABLEFORARRAY VALUES ('{4,5}', 8);
+INSERT INTO FKTABLEFORARRAY VALUES ('{4,4}', 9);
+INSERT INTO FKTABLEFORARRAY VALUES (NULL, 10);
+INSERT INTO FKTABLEFORARRAY VALUES ('{}', 11);
+INSERT INTO FKTABLEFORARRAY VALUES ('{1,NULL}', 12);
+INSERT INTO FKTABLEFORARRAY VALUES ('{NULL}', 13);
+INSERT INTO FKTABLEFORARRAYMDIM VALUES ('{{4,5},{1,2},{1,3}}', 14);
+INSERT INTO FKTABLEFORARRAYMDIM VALUES ('{{4,5},{NULL,2},{NULL,3}}', 15);
+
+-- Insert failed rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY VALUES ('{6}', 16);
+INSERT INTO FKTABLEFORARRAY VALUES ('{4,6}', 17);
+INSERT INTO FKTABLEFORARRAY VALUES ('{6,NULL}', 18);
+INSERT INTO FKTABLEFORARRAY VALUES ('{6,NULL,4,NULL}', 19);
+INSERT INTO FKTABLEFORARRAYMDIM VALUES ('{{1,2},{6,NULL}}', 20);
+INSERT INTO FKTABLEFORARRAYNOTNULL VALUES (NULL, 21);
+
+-- Check FKTABLE
+SELECT * FROM FKTABLEFORARRAY;
+
+-- Delete a row from PK TABLE (must fail due to ON DELETE NO ACTION)
+DELETE FROM PKTABLEFORARRAY WHERE ptest1=1;
+
+-- Check FKTABLE for removal of matched row
+SELECT * FROM FKTABLEFORARRAY;
+
+-- Update a row from PK TABLE (must fail due to ON UPDATE NO ACTION)
+UPDATE PKTABLEFORARRAY SET ptest1=7 WHERE ptest1=1;
+
+-- Check FKTABLE for update of matched row
+SELECT * FROM FKTABLEFORARRAY;
+
+-- Check UPDATE on FKTABLE
+UPDATE FKTABLEFORARRAY SET ftest1=ARRAY[1] WHERE ftest2=4;
+
+-- Check FKTABLE for update of matched row
+SELECT * FROM FKTABLEFORARRAY;
+
+DROP TABLE FKTABLEFORARRAY;
+DROP TABLE FKTABLEFORARRAYNOTNULL;
+DROP TABLE FKTABLEFORARRAYMDIM;
+
+-- Allowed references with actions (NO ACTION, RESTRICT)
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE NO ACTION ON UPDATE NO ACTION, ftest2 int );
+DROP TABLE FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE NO ACTION ON UPDATE RESTRICT, ftest2 int );
+DROP TABLE FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE RESTRICT ON UPDATE NO ACTION, ftest2 int );
+DROP TABLE FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE RESTRICT ON UPDATE RESTRICT, ftest2 int );
+DROP TABLE FKTABLEFORARRAY;
+-- Not allowed references (SET NULL, SET DEFAULT, CASCADE)
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE NO ACTION ON UPDATE SET DEFAULT, ftest2 int );
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE NO ACTION ON UPDATE SET NULL, ftest2 int );
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE RESTRICT ON UPDATE SET DEFAULT, ftest2 int );
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE RESTRICT ON UPDATE SET NULL, ftest2 int );
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE CASCADE ON UPDATE NO ACTION, ftest2 int );
+DROP TABLE FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE CASCADE ON UPDATE RESTRICT, ftest2 int );
+DROP TABLE FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE CASCADE ON UPDATE SET DEFAULT, ftest2 int );
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE CASCADE ON UPDATE SET NULL, ftest2 int );
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET DEFAULT ON UPDATE NO ACTION, ftest2 int );
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET DEFAULT ON UPDATE RESTRICT, ftest2 int );
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET DEFAULT ON UPDATE SET DEFAULT, ftest2 int );
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET DEFAULT ON UPDATE SET NULL, ftest2 int );
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET NULL ON UPDATE NO ACTION, ftest2 int );
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET NULL ON UPDATE RESTRICT, ftest2 int );
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET NULL ON UPDATE SET DEFAULT, ftest2 int );
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET NULL ON UPDATE SET NULL, ftest2 int );
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE NO ACTION ON UPDATE CASCADE, ftest2 int );
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE RESTRICT ON UPDATE CASCADE, ftest2 int );
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE CASCADE ON UPDATE CASCADE, ftest2 int );
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET DEFAULT ON UPDATE CASCADE, ftest2 int );
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+CREATE TABLE FKTABLEFORARRAY ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON DELETE SET NULL ON UPDATE CASCADE, ftest2 int );
+DROP TABLE IF EXISTS FKTABLEFORARRAY;
+
+-- Cleanup
+DROP TABLE PKTABLEFORARRAY;
+
+-- Check reference on empty table
+CREATE TABLE PKTABLEFORARRAY (ptest1 int PRIMARY KEY);
+CREATE TABLE FKTABLEFORARRAY  (ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY);
+INSERT INTO FKTABLEFORARRAY VALUES ('{}');
+DROP TABLE FKTABLEFORARRAY;
+DROP TABLE PKTABLEFORARRAY;
+
+-- Repeat a similar test using CHAR(1) keys rather than INTEGER
+CREATE TABLE PKTABLEFORARRAY ( ptest1 CHAR(1) PRIMARY KEY, ptest2 text );
+
+-- Populate the primary table
+INSERT INTO PKTABLEFORARRAY VALUES ('A', 'Test A');
+INSERT INTO PKTABLEFORARRAY VALUES ('B', 'Test B');
+INSERT INTO PKTABLEFORARRAY VALUES ('C', 'Test C');
+
+-- Create the refrencing table
+CREATE TABLE FKTABLEFORARRAY ( ftest1 char(1)[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAY ON UPDATE RESTRICT ON DELETE RESTRICT, ftest2 int );
+
+-- Insert valid rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY VALUES ('{"A"}', 1);
+INSERT INTO FKTABLEFORARRAY VALUES ('{"B"}', 2);
+INSERT INTO FKTABLEFORARRAY VALUES ('{"C"}', 3);
+INSERT INTO FKTABLEFORARRAY VALUES ('{"A","B","C"}', 4);
+
+-- Insert invalid rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY VALUES ('{"D"}', 5);
+INSERT INTO FKTABLEFORARRAY VALUES ('{"A","B","D"}', 6);
+
+-- Check FKTABLE
+SELECT * FROM FKTABLEFORARRAY;
+
+-- Delete a row from PK TABLE (must fail due to ON DELETE RESTRICT)
+DELETE FROM PKTABLEFORARRAY WHERE ptest1='A';
+
+-- Check FKTABLE for removal of matched row
+SELECT * FROM FKTABLEFORARRAY;
+
+-- Update a row from PK TABLE (must fail due to ON UPDATE RESTRICT)
+UPDATE PKTABLEFORARRAY SET ptest1='D' WHERE ptest1='B';
+
+-- Check FKTABLE for update of matched row
+SELECT * FROM FKTABLEFORARRAY;
+
+-- Cleanup
+DROP TABLE FKTABLEFORARRAY;
+DROP TABLE PKTABLEFORARRAY;
+
+-- Composite primary keys
+CREATE TABLE PKTABLEFORARRAY ( id1 CHAR(1), id2 CHAR(1), ptest2 text, PRIMARY KEY (id1, id2) );
+
+-- Populate the primary table
+INSERT INTO PKTABLEFORARRAY VALUES ('A', 'A', 'Test A');
+INSERT INTO PKTABLEFORARRAY VALUES ('A', 'B', 'Test B');
+INSERT INTO PKTABLEFORARRAY VALUES ('B', 'C', 'Test B');
+
+-- Create the refrencing table
+CREATE TABLE FKTABLEFORARRAY ( fid1 CHAR(1), fid2 CHAR(1)[], ftest2 text, FOREIGN KEY (fid1, EACH ELEMENT OF fid2) REFERENCES PKTABLEFORARRAY);
+
+-- Insert valid rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY VALUES ('A', ARRAY['A','B'], '1');
+INSERT INTO FKTABLEFORARRAY VALUES ('B', ARRAY['C'], '2');
+
+-- Insert invalid rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY VALUES ('A', ARRAY['A','B', 'C'], '3');
+INSERT INTO FKTABLEFORARRAY VALUES ('B', ARRAY['A'], '4');
+
+-- Cleanup
+DROP TABLE FKTABLEFORARRAY;
+DROP TABLE PKTABLEFORARRAY;
+
+-- Test Array Element Foreign Keys with composite type
+CREATE TYPE INVOICEID AS (year_part INTEGER, progressive_part INTEGER);
+CREATE TABLE PKTABLEFORARRAY ( id INVOICEID PRIMARY KEY,  ptest2 text);
+
+-- Populate the primary table
+INSERT INTO PKTABLEFORARRAY VALUES (ROW(2010, 99), 'Last invoice for 2010');
+INSERT INTO PKTABLEFORARRAY VALUES (ROW(2011, 1), 'First invoice for 2011');
+INSERT INTO PKTABLEFORARRAY VALUES (ROW(2011, 2), 'Second invoice for 2011');
+
+-- Create the refrencing table
+CREATE TABLE FKTABLEFORARRAY ( id SERIAL PRIMARY KEY, invoice_ids INVOICEID[], FOREIGN KEY (EACH ELEMENT OF invoice_ids) REFERENCES PKTABLEFORARRAY, ftest2 TEXT );
+
+-- Insert valid rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY(invoice_ids, ftest2) VALUES (ARRAY['(2010,99)']::INVOICEID[], 'Product A');
+INSERT INTO FKTABLEFORARRAY(invoice_ids, ftest2) VALUES (ARRAY['(2011,1)','(2011,2)']::INVOICEID[], 'Product B');
+INSERT INTO FKTABLEFORARRAY(invoice_ids, ftest2) VALUES (ARRAY['(2011,2)']::INVOICEID[], 'Product C');
+
+-- Insert invalid rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY(invoice_ids, ftest2) VALUES (ARRAY['(2011,99)']::INVOICEID[], 'Product A');
+INSERT INTO FKTABLEFORARRAY(invoice_ids, ftest2) VALUES (ARRAY['(2011,1)','(2010,1)']::INVOICEID[], 'Product B');
+
+-- Check FKTABLE
+SELECT * FROM FKTABLEFORARRAY;
+
+-- Delete a row from PK TABLE
+DELETE FROM PKTABLEFORARRAY WHERE id=ROW(2010,99);
+
+-- Check FKTABLE for removal of matched row
+SELECT * FROM FKTABLEFORARRAY;
+
+-- Update a row from PK TABLE
+UPDATE PKTABLEFORARRAY SET id=ROW(2011,99) WHERE id=ROW(2011,1);
+
+-- Check FKTABLE for update of matched row
+SELECT * FROM FKTABLEFORARRAY;
+
+-- Cleanup
+DROP TABLE FKTABLEFORARRAY;
+DROP TABLE PKTABLEFORARRAY;
+DROP TYPE INVOICEID;
+
+-- Check for an array column referencing another array column (NOT ELEMENT FOREIGN KEY)
+-- Create primary table with an array primary key
+CREATE TABLE PKTABLEFORARRAY ( id INT[] PRIMARY KEY,  ptest2 text);
+
+-- Create the refrencing table
+CREATE TABLE FKTABLEFORARRAY ( id SERIAL PRIMARY KEY, fids INT[] REFERENCES PKTABLEFORARRAY, ftest2 TEXT );
+
+-- Populate the primary table
+INSERT INTO PKTABLEFORARRAY VALUES ('{1,1}', 'A');
+INSERT INTO PKTABLEFORARRAY VALUES ('{1,2}', 'B');
+
+-- Insert valid rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY (fids, ftest2) VALUES ('{1,1}', 'Product A');
+INSERT INTO FKTABLEFORARRAY (fids, ftest2) VALUES ('{1,2}', 'Product B');
+
+-- Insert invalid rows into FK TABLE
+INSERT INTO FKTABLEFORARRAY (fids, ftest2) VALUES ('{0,1}', 'Product C');
+INSERT INTO FKTABLEFORARRAY (fids, ftest2) VALUES ('{2,1}', 'Product D');
+
+-- Cleanup
+DROP TABLE FKTABLEFORARRAY;
+DROP TABLE PKTABLEFORARRAY;
+
+-- ---------------------------------------
+-- Multi-column "ELEMENT" foreign key tests
+-- ---------------------------------------
+
+-- Create DIM1 table with two-column primary key
+CREATE TABLE DIM1 (X INTEGER NOT NULL, Y INTEGER NOT NULL, PRIMARY KEY (X, Y));
+-- Populate DIM1 table pairs
+INSERT INTO DIM1 SELECT x.t, x.t * y.t
+	FROM (SELECT generate_series(1, 10) AS t) x,
+	(SELECT generate_series(0, 10) AS t) y;
+
+
+-- Test with TABLE declaration of an element foreign key constraint (NO ACTION)
+CREATE TABLE F1 (
+	x INTEGER PRIMARY KEY, y INTEGER[],
+	FOREIGN KEY (x, EACH ELEMENT OF y) REFERENCES DIM1(x, y)
+);
+-- Insert facts
+INSERT INTO F1 VALUES (1, '{0,1,2,3,4,5}'); -- OK
+INSERT INTO F1 VALUES (2, '{0,2,4,6}'); -- OK
+INSERT INTO F1 VALUES (3, '{0,3,6,9,0,3,6,9,0,0,0,0,9,9}'); -- OK (multiple occurrences)
+INSERT INTO F1 VALUES (4, '{0,2,4}'); -- FAILS (2 is not present)
+INSERT INTO F1 VALUES (4, '{0,NULL,4}'); -- OK
+INSERT INTO F1 VALUES (5, '{0,NULL,5}'); -- OK
+-- Try updates
+UPDATE F1 SET y = '{0,2,4,6}' WHERE x = 2; -- OK
+UPDATE F1 SET y = '{0,2,3,4,6}' WHERE x = 2; -- FAILS
+UPDATE F1 SET x = 20, y = '{0,2,3,4,6}' WHERE x = 2; -- FAILS (20 does not exist)
+UPDATE F1 SET y = '{0,4,8}' WHERE x = 4; -- OK
+UPDATE F1 SET y = '{0,5,NULL,10}' WHERE x = 5; -- OK
+DROP TABLE F1;
+
+
+-- Test with FOREIGN KEY after TABLE population
+CREATE TABLE F1 (
+	x INTEGER PRIMARY KEY, y INTEGER[]
+);
+-- Insert facts
+INSERT INTO F1 VALUES (1, '{0,1,2,3,4,5}'); -- OK
+INSERT INTO F1 VALUES (2, '{0,2,4,6}'); -- OK
+INSERT INTO F1 VALUES (3, '{0,3,6,9,0,3,6,9,0,0,0,0,9,9}'); -- OK (multiple occurrences)
+INSERT INTO F1 VALUES (4, '{0,2,4}'); -- OK (2 is not present)
+INSERT INTO F1 VALUES (5, '{0,NULL,5}'); -- OK
+-- Add foreign key (FAILS)
+ALTER TABLE F1 ADD FOREIGN KEY (x, EACH ELEMENT OF y) REFERENCES DIM1(x, y);
+DROP TABLE F1;
+
+
+-- Test with TABLE declaration of a two-dim ELEMENT foreign key constraint (FAILS)
+CREATE TABLE F1 (
+	x INTEGER[] PRIMARY KEY, y INTEGER[],
+	FOREIGN KEY (EACH ELEMENT OF x, EACH ELEMENT OF y) REFERENCES DIM1(x, y)
+);
+
+
+-- Test with two-dim ELEMENT foreign key after TABLE population
+CREATE TABLE F1 (
+	x INTEGER[] PRIMARY KEY, y INTEGER[]
+);
+INSERT INTO F1 VALUES ('{1}', '{0,1,2,3,4,5}'); -- OK
+INSERT INTO F1 VALUES ('{1,2}', '{0,2,4,6}'); -- OK
+-- Add foreign key (FAILS)
+ALTER TABLE F1 ADD FOREIGN KEY (EACH ELEMENT OF x, EACH ELEMENT OF y) REFERENCES DIM1(x, y);
+DROP TABLE F1;
+
+-- Cleanup
+DROP TABLE DIM1;
+
+
+-- Check for potential name conflicts (with internal integrity checks)
+CREATE TABLE x1(x1 int, x2 int, PRIMARY KEY(x1,x2));
+INSERT INTO x1 VALUES
+       (1,4),
+       (1,5),
+       (2,4),
+       (2,5),
+       (3,6),
+       (3,7)
+;
+CREATE TABLE x2(x1 int[], x2 int, FOREIGN KEY(EACH ELEMENT OF x1, x2) REFERENCES x1);
+INSERT INTO x2 VALUES ('{1,2}',4);
+INSERT INTO x2 VALUES ('{1,3}',6); -- FAILS
+DROP TABLE x2;
+CREATE TABLE x2(x1 int[], x2 int);
+INSERT INTO x2 VALUES ('{1,2}',4);
+INSERT INTO x2 VALUES ('{1,3}',6);
+ALTER TABLE x2 ADD CONSTRAINT fk_const FOREIGN KEY(EACH ELEMENT OF x1, x2) REFERENCES x1;  -- FAILS
+DROP TABLE x2;
+DROP TABLE x1;
+
+
+-- ---------------------------------------
+-- Multi-dimensional "ELEMENT" foreign key tests
+-- ---------------------------------------
+
+-- Create DIM1 table with two-column primary key
+CREATE TABLE DIM1 (X INTEGER NOT NULL PRIMARY KEY,
+	CODE TEXT NOT NULL UNIQUE);
+-- Populate DIM1 table pairs
+INSERT INTO DIM1 SELECT t, 'DIM1-' || lpad(t::TEXT, 2, '0')
+	FROM (SELECT generate_series(1, 10)) x(t);
+
+-- Test with TABLE declaration of an element foreign key constraint (NO ACTION)
+CREATE TABLE F1 (
+	ID SERIAL PRIMARY KEY,
+	SLOTS INTEGER[3][3], FOREIGN KEY (EACH ELEMENT OF SLOTS) REFERENCES DIM1
+);
+INSERT INTO F1(SLOTS) VALUES ('{{NULL, 1, NULL}, {NULL, NULL, 3}, {NULL, NULL, 6}}'); -- OK
+INSERT INTO F1(SLOTS) VALUES ('{{NULL, 1, NULL}, {NULL, NULL, 11}, {NULL, NULL, 6}}'); -- FAILS
+INSERT INTO F1(SLOTS) VALUES ('{{1, 2, 3}, {4, 5, 6}, {7, 8, 9}}'); -- OK
+INSERT INTO F1(SLOTS) VALUES ('{1, 2, 3, 4, 5, 6, 7, 8, 9}'); -- OK
+UPDATE F1 SET SLOTS = '{{NULL, 1, NULL}, {NULL, NULL, 3}, {7, 8, 10}}' WHERE ID = 1; -- OK
+UPDATE F1 SET SLOTS = '{{100, 100, 100}, {NULL, NULL, 20}, {7, 8, 10}}' WHERE ID = 1; -- FAILS
+DROP TABLE F1;
+
+-- Test with postponed foreign key
+CREATE TABLE F1 (
+	ID SERIAL PRIMARY KEY,
+	SLOTS INTEGER[3][3]
+);
+INSERT INTO F1(SLOTS) VALUES ('{{NULL, 1, NULL}, {NULL, NULL, 3}, {NULL, NULL, 6}}'); -- OK
+INSERT INTO F1(SLOTS) VALUES ('{{NULL, 1, NULL}, {NULL, NULL, 11}, {NULL, NULL, 6}}'); -- OK
+INSERT INTO F1(SLOTS) VALUES ('{{1, 2, 3}, {4, 5, 6}, {7, 8, 9}}'); -- OK
+INSERT INTO F1(SLOTS) VALUES ('{1, 2, 3, 4, 5, 6, 7, 8, 9}'); -- OK
+ALTER TABLE F1 ADD FOREIGN KEY (EACH ELEMENT OF SLOTS) REFERENCES DIM1; -- FAILS
+DELETE FROM F1 WHERE ID = 2; -- REMOVE ISSUE
+ALTER TABLE F1 ADD FOREIGN KEY (EACH ELEMENT OF SLOTS) REFERENCES DIM1; -- NOW OK
+INSERT INTO F1(SLOTS) VALUES ('{{NULL, 1, NULL}, {NULL, NULL, 11}, {NULL, NULL, 6}}'); -- FAILS
+DROP TABLE F1;
+
+-- Leave tables in the database
+CREATE TABLE PKTABLEFORELEMENTFK ( ptest1 int PRIMARY KEY, ptest2 text );
+CREATE TABLE FKTABLEFORELEMENTFK ( ftest1 int[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORELEMENTFK, ftest2 int );
+
+-- Check ALTER TABLE ALTER TYPE
+ALTER TABLE FKTABLEFORELEMENTFK ALTER FTEST1 TYPE INT[];
+
+-- Check GIN index
+-- Define PKTABLEFORARRAYGIN
+CREATE TABLE PKTABLEFORARRAYGIN ( ptest1 int PRIMARY KEY, ptest2 text );
+
+-- Insert test data into PKTABLEFORARRAYGIN
+INSERT INTO PKTABLEFORARRAYGIN VALUES (1, 'Test1');
+INSERT INTO PKTABLEFORARRAYGIN VALUES (2, 'Test2');
+INSERT INTO PKTABLEFORARRAYGIN VALUES (3, 'Test3');
+INSERT INTO PKTABLEFORARRAYGIN VALUES (4, 'Test4');
+INSERT INTO PKTABLEFORARRAYGIN VALUES (5, 'Test5');
+INSERT INTO PKTABLEFORARRAYGIN VALUES (6, 'Test6');
+
+-- Define FKTABLEFORARRAYGIN
+CREATE TABLE FKTABLEFORARRAYGIN ( ftest1 int[],
+    ftest2 int PRIMARY KEY,
+    FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORARRAYGIN
+    ON DELETE NO ACTION ON UPDATE NO ACTION);
+
+-- -- Create index on FKTABLEFORARRAYGIN
+CREATE INDEX ON FKTABLEFORARRAYGIN USING gin (ftest1 array_ops);
+
+-- Populate Table
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{5}', 1);
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{3,2}', 2);
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{3,5,2,5}', 3);
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{3,4,4}', 4);
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{3,5,4,1,3}', 5);
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{1}', 6);
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{5,1}', 7);
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{2,1,2,4,1}', 8);
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{4,2}', 9);
+INSERT INTO FKTABLEFORARRAYGIN VALUES ('{3,4,5,3}', 10);
+
+-- Try UPDATE
+UPDATE PKTABLEFORARRAYGIN SET ptest1=7 WHERE ptest1=6;
+
+-- Try using the indexable operator
+SELECT * FROM FKTABLEFORARRAYGIN WHERE ftest1 @>> 5;
+
+-- Cleanup
+DROP TABLE FKTABLEFORARRAYGIN;
+DROP TABLE PKTABLEFORARRAYGIN;
+
+-- ---------------------------------------
+-- Invalid refrencing key tests
+-- ---------------------------------------
+CREATE TABLE PKTABLEVIOLATING ( ptest1 int PRIMARY KEY, ptest2 text );
+
+-- Attempt fk constraint between int <-> int
+CREATE TABLE FKTABLEVIOLATING ( ftest1 int, FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORELEMENTFK, ftest2 int );
+
+-- Attempt fk constraint between int <-> char[]
+CREATE TABLE FKTABLEVIOLATING ( ftest1 char[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORELEMENTFK, ftest2 int );
+
+-- Attempt fk constraint between int <-> char
+CREATE TABLE FKTABLEVIOLATING ( ftest1 char, FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORELEMENTFK, ftest2 int );
+
+-- Attempt fk constraint between int <-> smallint[]
+CREATE TABLE FKTABLEVIOLATING ( ftest1 smallint[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORELEMENTFK, ftest2 int );
+
+-- Attempt fk constraint between int <-> bigint[]
+CREATE TABLE FKTABLEVIOLATING ( ftest1 bigint[], FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORELEMENTFK, ftest2 int );
+
+-- Attempt fk constraint between int <-> int2vector
+CREATE TABLE FKTABLEVIOLATING ( ftest1 int2vector, FOREIGN KEY (EACH ELEMENT OF ftest1) REFERENCES PKTABLEFORELEMENTFK, ftest2 int );
```
