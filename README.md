
# 🤔 Problems With Duplicate Data? 

It is inevitable that every developer will deal with duplicate data at some point in their career. In my 30+ years as a developer, I have encountered this issue numerous times. Here is an example that identifies and eliminates duplicates using the OLAP specifications ROW_NUMBER, OVER, and PARTITION.

Firstly, we need to ask ourselves these questions.

1. What columns constitues duplicate data?
2. What is the primary key of the table?

I work with many transaction-based files that use primary keys such as txn_id. That provides something to uniquely id rows in the table. In some cases, though, this is impossible, especially if dealing with legacy files. In those cases, we can utilize the relative record number which is what I do in the example below.

The test library I am using is TESTBB.

## Example 

1. Create a test table for the demo. 

    ```sql
    create table testbb.my_test_table (
        field_1  smallint   not null with default,
        field_2  char(20)   not null with default,
        field_3  char(10)   not null with default);
    ```
    I'm not using system names but one will be generated. It should be named MY_TE00001 since the name will consist of the first 5 characters of the table name plus a 5 character sequence number. So you can use green-screen tools to acces the table.
<br><br>

2. Put some test data into the table.

    ```sql
    insert into testbb.my_test_table values
        (1, 'This is record 1', 'BBROCK'),
        (2, 'This is record 2', 'AJONES'),
        (2, 'This is record 2', 'AJONES'), 
        (2, 'This is record 2', 'AJONES'),
        (3, 'This is record 3', 'HHUBB'), 
        (3, 'This is record 3', 'HHUBB'), 
        (4, 'This is record 4', 'JDUDE'),
        (5, 'This is record 5', 'LJOHN'),
        (5, 'This is record 5', 'LJOHN');

    ```
    In this example, all the columns in the row are duplicated. This might not always be the case especially with date and time columns.
    <br><br>

3. View the data using the OLAP specifications.

    ```sql
    select row_number() over (partition by field_1 order by field_1) 
        as row_id, field_1, field_2 
      from testbb.my_test_table;
    ```
    I only need field_1 to identify the duplicates. I could have used all 3 but in this case that just wasn't necessary. I chose field_1 and field_2 for the results table. Notice what the row_id is doing in the results table. We have partitioned by field_1. So it numbers the results in each partition! This is excellent because now we know that any row with an id > 1 is a duplicate. So how do we delete the records from our table? 

    <table>
        <tr>
            <th>row_id</th>
            <th>field_1</th>
            <th>field_2</th>
        </tr>
        <tr>
            <td>1</td>
            <td>1</td>
            <td>This is record 1</td>
        </tr>
        <tr>
            <td>1</td>
            <td>2</td>
            <td>This is record 2</td>
        </tr>
            <tr>
            <td>2</td>
            <td>2</td>
            <td>This is record 2</td>
        </tr>
            <tr>
            <td>3</td>
            <td>2</td>
            <td>This is record 2</td>
        </tr>
            <tr>
            <td>1</td>
            <td>3</td>
            <td>This is record 3</td>
        </tr>
            <tr>
            <td>2</td>
            <td>3</td>
            <td>This is record 3</td>
        </tr>
            <tr>
            <td>1</td>
            <td>4</td>
            <td>This is record 4</td>
        </tr>
            <tr>
            <td>1</td>
            <td>5</td>
            <td>This is record 5</td>
        </tr>
            <tr>
            <td>2</td>
            <td>5</td>
            <td>This is record 5</td>
        </tr> 
    </table>
<br>

4. Create a new table based on results from above step.

    ```sql
    create table testbb.my_test_table_dups as (
          select row_number() over (partition by field_1 order by field_1) 
              as row_id, rrn(testbb.my_test_table) as rrn, field_1, field_2 
            from testbb.my_test_table) with data;  
    ```
    Create a new table that will be used to id the rows in the original table and delete them. But we need a unique column and one doesn't exist! Well, we can use the relative record number for that purpose. The rrn gives us a unique id for each record in this new file.   

    <table>
        <tr>
            <th>row_id</th>
            <th>rrn</th>
            <th>field_1</th>
            <th>field_2</th>
        </tr>
        <tr>
            <td>1</td>
            <td>1</td>
            <td>1</td>
            <td>This is record 1</td>
        </tr>
        <tr>
            <td>1</td>
            <td>2</td>
            <td>2</td>
            <td>This is record 2</td>
        </tr>
            <tr>
            <td>2</td>
            <td>3</td>
            <td>2</td>
            <td>This is record 2</td>
        </tr>
            <tr>
            <td>3</td>
            <td>4</td>
            <td>2</td>
            <td>This is record 2</td>
        </tr>
            <tr>
            <td>1</td>
            <td>5</td>
            <td>3</td>
            <td>This is record 3</td>
        </tr>
            <tr>
            <td>2</td>
            <td>6</td>
            <td>3</td>
            <td>This is record 3</td>
        </tr>
            <tr>
            <td>1</td>
            <td>7</td>
            <td>4</td>
            <td>This is record 4</td>
        </tr>
            <tr>
            <td>1</td>
            <td>8</td>
            <td>5</td>
            <td>This is record 5</td>
        </tr>
            <tr>
            <td>2</td>
            <td>9</td>
            <td>5</td>
            <td>This is record 5</td>
        </tr> 
    </table>
<br>

5. Identify the duplicates in the original table.

    ```sql
     select * from testbb.my_test_table
      where rrn(testbb.my_test_table) in
            (select rrn 
               from testbb.my_test_table_dups
               where row_id > 1)
      order by field_1; 
    ```
    Using the rrn, we can zero in on the duplicates.   

    <table>
        <tr>
            <th>field_1</th>
            <th>field_2</th>
            <th>field_3</th>
        </tr>
        <tr>
            <td>2</td>
            <td>This is record 2</td>
            <td>AJONES</td>
        </tr>
        <tr>
            <td>2</td>
            <td>This is record 2</td>
            <td>AJONES</td>
        </tr>
        <tr>
            <td>3</td>
            <td>This is record 3</td>
            <td>HHUBB</td>
        </tr>
        <tr>
            <td>5</td>
            <td>This is record 5</td>
            <td>LJOHN</td>
        </tr>       
    </table>
<br>

6. Delete the duplicates in the original table.

    ```sql
     delete from testbb.my_test_table
      where rrn(testbb.my_test_table) in
            (select rrn 
               from testbb.my_test_table_dups
               where row_id > 1)
      order by field_1; 
    ```
    Just using the statement from the above step changing the "select *" to "delete".
<br><br>

7. Check the results.
    <table>
        <tr>
            <th>field_1</th>
            <th>field_2</th>
            <th>field_3</th>
        </tr>
        <tr>
            <td>1</td>
            <td>This is record 1</td>
            <td>BBROCK</td>
        </tr>
        <tr>
            <td>2</td>
            <td>This is record 2</td>
            <td>AJONES</td>
        </tr>
        <tr>
            <td>3</td>
            <td>This is record 3</td>
            <td>HHUBB</td>
        </tr>
        <tr>
            <td>4</td>
            <td>This is record 4</td>
            <td>JDUDE</td>
        </tr>       
        <tr>
            <td>5</td>
            <td>This is record 5</td>
            <td>LJOHN</td>
        </tr>       
    </table>
<br>

Nice and clean with no caffiene! 

I'm sure all of these individual statements could be combined into a single statement and run against the file in one fell swoop. In my experience, I prefer to break down big queries into smaller ones so that each step can be analyzed in greater detail. Larger tables may need a different approach. But hopefully, you gain some value from this example.

<img src="db2.png" width="10%" align="center"> Dig the data!

<!--
**bkbrock59/bkbrock59** is a ✨ _special_ ✨ repository because its `README.md` (this file) appears on your GitHub profile.

Here are some ideas to get you started:

- 🔭 I’m currently working on ...
- 🌱 I’m currently learning ...
- 👯 I’m looking to collaborate on ...
- 🤔 I’m looking for help with ...
- 💬 Ask me about ...
- 📫 How to reach me: ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...
-->
