---
title: "Processing Database Data with Python"
date: 2019-06-30
translationKey: "posts/python-blob-to-binary"
categories: 
  - python
image: "../../images/python.webp"
tags:
  - oracle
  - postgresql
  - blob
  - python
---

I didn't have much interest in databases because I didn't know much about them and thought it was a little distant from creating new programs, but it's difficult not to interact with databases while working in the IT industry. And the DB who interacts with them in this way becomes a less friendly problem. In this job, I mainly wrote scripts using Python, but the main task was to collaborate with the DB. So in this post, I would like to talk about how I used Python to connect to a DB, issue SQL statements, and handle the data.

The task I was given this time was to create a script that extracts binary data from a DB and uploads it to AWS S3[^1]. Also, the table needs to be updated when the upload is finished, but the DB to be extracted and the DB to be updated are different. One is Oracle and the other is PostgreSQL. There were also conditions such as installing a logger so that when a connection to each DB or S3 fails, it can be seen in the log, and reading DB connection information from an external file. I talked about the logger in [a previous post](../python-logger/), so I will omit it this time. There are other conditions to extract binary data, such as receiving an argument, checking the last updated date and saving files after that, determining the path name from the column of the table when uploading to S3, and updating the last updated date after uploading the image.

To summarize, the specifications of the script we will create this time are as follows.

1. receive arguments on command line
2. Refer to PostgreSQL table (1) from the argument and get the last update date of the work corresponding to the argument.
3. SELECT binary data after the last update date from OcaleDB table and save it as a JPG file
4. Obtain the column that is the path to save the file from the PostgreSQL table (2).
5. Upload images to S3 in folders for each JPG file
6. Record the information of the JPG file uploaded to PostgreSQL table (3)
7. PostgreSQL table (1) Update last update date
8. If it completes normally, the exit code will be 0, and if an exception occurs, it will output the exit code 9 and exit.

In the case of PostgreSQL, there are many tables to reference and there is a processing order, so it seems a little complicated, but I decided to try linking with the DB and create a script first.

## Link DB with Python

Surprisingly, connecting to a DB with Python wasn't that difficult. Each uses different modules and has different commands, but when I searched for them, I found many examples. For Oracle, prepare the host name, port, service name, user name, and password. The module used is `cx_Oracle`.

For PostgreSQL, use `psycopg2`. The information required for connection is host name, port, DB name, user name, and password. However, both modules can be installed from `pip`, but in the case of psycopg2, it has library dependencies, so you need to install `postgresql-develop` from yum based on Amazon Linux. I don't know much about the others since I haven't touched them, but I think it's the same for CentOS. If you have any problems installing psycopg2, check the error message and install the necessary PosgreSQL libraries.

Although the steps to connect to the two DBs are slightly different, the processing after connection is the same (actually, the syntax of the SQL statements also seems to be slightly different...). Now, let's separately explain how to connect Oracle and PostgreSQL and the processing after connection.

## Connection (for Oracle)

```python
import cx_Oracle

tns = cx_Oracle.makedsn('host', 'port', 'service_name')
connect = cx_Oracle.connect('user_name', 'password', tns)
```

Enter the host name, port, and service name in `makedsn` of cx_Oracle. Then enter that information along with your username and password when using `connect`. This completes the Oracle connection settings.

## Connection (for PostgreSQL)

```python
import psycopg2

connect = psycopg2.connect('host=' + 'host' + ' port=' + 'port' + ' dbname=' + 'db_name' + ' user=' + 'user_name' + ' password=' + 'password')
```

For PostgreSQL it's easier. Just like `psql` of the command line tools used in Linux, all you have to do is line up the information necessary for the connection as a string and type `connect`.

## Common parts of DB processing

```python
# Connect to the DB and get a cursor
cursor = connect.cursor()
# Execute SQL with the cursor
cursor.execute('SQL you want to run')

# Get the result of the SQL statement
# When you need only one row
result = cursor.fetchone()
# When you want to fetch and process 1,000 rows at a time
result = cursor.fetchmany(1000)
# When you want to fetch everything
result = cursor.fetchall()

# Close the cursor
cursor.close()
# Commit the SQL statement (for INSERT/UPDATE and so on)
connect.commit()
# Close the connection
connect.close()
```

After connecting, obtain a cursor and issue SQL statements with that cursor. `fecth` is used to obtain the execution results of SQL statements, but there are three options depending on the scale of the results you want to process. Also, if you want to process a large amount of data, it seems better to specify the fetch size with `fetchmany` rather than `fetchall`. Too much data will affect performance. In Python, the result obtained after fetching is an array of one record and a list of them. In code, it would look like this:

```python
# For 'SELECT column_1, column_2 FROM TABLE'

result = cursor.fetchmany(1000)

for row in result: # The result is a list of 1,000 rows
    print row[0] # Prints column_1
    print row[1] # Prints column_2

```

This completes the basics of DB linkage. Now, I will show you how to publish the script code.

## Code to upload image file to S3

```python
# -*- coding: UTF-8 -*-
import os, sys, cx_Oracle, psycopg2, json, boto3, datetime, shutil

function_code = 'received_from_argument'

# DB environment settings (read from DBConnection.conf)
HOST_POST = 'PostgreSQL host'
PORT_POST = 'PostgreSQL port'
DB_NAME_POST = 'PostgreSQL DB name'
USER_POST = 'PostgreSQL user'
PWD_POST = 'PostgreSQL password'
HOST_ORAC = 'Oracle host'
PORT_ORAC = 'Oracle port'
SERVICE_NAME = 'Oracle service name'
USER_ORAC = 'Oracle schema'
PWD_ORAC = 'Oracle password'

# Command-line arguments
args = sys.argv

# Environment-related variables
imageFolder = '/tmp/images'

# Pre-processing for incremental synchronization
def GetProcdate(args):
    global function_code
    function_code = args[1]
 
    try:
        # Connect to PostgreSQL
        print ('>> Starting Job. The function Code is: ' + function_code)
        connect = psycopg2.connect('host=' + HOST_POST + ' port=' + PORT_POST + ' dbname=' + DB_NAME_POST + ' user=' + USER_POST + ' password=' + PWD_POST)
        cursor = connect.cursor()
    except:
        print ('>> Unable to connect PostgreSQL. quitting.')
        sys.exit(9)

    selectSQL = "SELECT last_date FROM date_table WHERE function_code='%s'" % (function_code,)
    cursor.execute(selectSQL)
    result = cursor.fetchone()

    # If there is a result, store the last update date in a variable
    if (result is not None):
        last_date = result[0]
    else:
        print ('>> No data matches with ' + function_code + '. quitting.')
        sys.exit(1)

    cursor.close()
    connect.close()

    # Continue to the next step
    GetImageFromTable(last_date)

# Run SQL against Oracle and read/save the files
def GetImageFromTable(last_date):
    global imageFolder
    try:
        # Start the Oracle connection
        tns = cx_Oracle.makedsn(HOST_ORAC, PORT_ORAC, SERVICE_NAME)
        connect = cx_Oracle.connect(USER_ORAC, PWD_ORAC, tns)
        cursor = connect.cursor()
    except:
        print ('>> Unable to connect Oracle DB. quitting.')
        sys.exit(9)

    # Get product image files from the Oracle table
    cursor.execute("SELECT image_data, image_name FROM WHERE >=(:last_date)", {'last_date': last_date})

    # Create the folder if it does not exist; otherwise remove it first
    if os.path.exists(imageFolder):
        shutil.rmtree(imageFolder)
    else:
        os.mkdir(imageFolder)

    # Fetch 1,000 rows at a time
    while True:
        rows = cursor.fetchmany(1000)
        
        # Exit the loop if there are no results
        if len(rows) == 0:
            break

        # Save image files into the local image directory (image_name.jpg)
        for image in rows:
            fileNameS = imageFolder + '/' + str(image[1]) + '.jpg'
            imageFile = open(fileNameS, 'wb+')
            imageFile.write(image[0].read())
            imageFile.close()
            counter = counter + 1

    # Count how many files were saved
    fileCount = len([name for name in os.listdir(imageFolder) if os.path.isfile(os.path.join(imageFolder, name))])
    print ('>> As result: ' + str(fileCount) + ' files were written.')
    print ('>> Finished job. Closing connection.')
    cursor.close()
    connect.close()
    FileUploadToS3()

# Store image files in S3
def FileUploadToS3():
    # Read global variables
    global imageFolder
    global function_code

    # Get the list of file names (image names)
    files = os.listdir(imageFolder)
    item_cd_list = [os.path.splitext(f)[0] for f in files if os.path.isfile(os.path.join(imageFolder, f))]

    try:
        # Connect to PostgreSQL
        connect = psycopg2.connect('host=' + HOST_POST + ' port=' + PORT_POST + ' dbname=' + DB_NAME_POST + ' user=' + USER_POST + ' password=' + PWD_POST)
        cursor = connect.cursor()
    except:
        print ('>> Unable to connect PostgreSQL. quitting.')
        sys.exit(9)

    # Counter used to track processed files
    fileCount = 0

    for item_cd in item_cd_list:
        # Get the record matching the code from the item table to build the storage path
        selectSQL1 = "SELECT file_path1, file_path2, file_path3 FROM item_table WHERE item_cd='%s'" % (str(item_cd),)
        cursor.execute(selectSQL1)
        resultMaster = cursor.fetchone()

        # Continue even if no matching record is found
        if resultMaster is None:
        print ('>> Result was 0. continue job.')
            continue

        # Extract the required data from the result and store it in variables
        image_id = str(resultMaster[2])
        path = str(resultMaster[0]) + '/' + str(resultMaster[1]) + '/' + str(resultMaster[2]) + '/'

        # Get the matching record from the image-management table for branching logic
        selectSQL2 = "SELECT * FROM WHERE image_table image_id='%s';" % (image_id,)
        cursor.execute(selectSQL2)
        result = cursor.fetchone()
        uploadToS3(item_cd, path)
        fileCount = fileCount + 1
        print ('>> Processing file upload.')

    # Update the job completion timestamp
    updateProcSQL = "UPDATE date_table SET last_date = CURRENT_TIMESTAMP WHERE function_code='%s'" % (function_code,)
    cursor.execute(updateProcSQL)

    # Finish the job
    print ('>> As result: ' + str(fileCount) + ' files were uploaded.')
    print ('>> Finished job. Closing connection and quit.')
    cursor.close()
    connect.commit()
    connect.close()
    sys.exit(0)

# Upload image files to S3
def uploadToS3(item_cd, path):
    global imageFolder
    print ('>> Upload image files to AWS S3.')
    bucket_name = 'image'
    s3 = boto3.resource('s3')

    try:
        # Save the file stored on the server to the specified path
        s3.Bucket(bucket_name).upload_file(imageFolder + '/' + item_cd + '.jpg', path + '/1_1_' + datetime.datetime.today().strftime('%Y%m%d%H%M%S') + '.jpg')
        print ('>> Image file uploaded. (Item code: ' + item_cd + ')')
    except:
        print ('>> Unable to upload files. quitting.')
        sys.exit(9)

# Read connection information from an input file
def getDBConnection():

    # Get the path to the DBConnection settings file
    pwd = os.path.dirname(os.path.abspath(__file__))
    dbConnection = pwd.rsplit("/", 1)[0] + '/env/DBConnection.conf'

    # Exit with an error if the file does not exist
    if (not os.path.exists(dbConnection)):
        print ('>> Please check DBConnection.conf. quitting.')
        sys.exit(9)

    # Read the file
    connectionInfo = open(dbConnection, 'r')
    lines = connectionInfo.readlines()

    # Declare the variables so they can be stored globally
    global HOST_POST
    global PORT_POST
    global DB_NAME_POST
    global USER_POST
    global PWD_POST
    global HOST_ORAC
    global PORT_ORAC
    global SERVICE_NAME
    global USER_ORAC
    global PWD_ORAC

    # Store values when matching data is found
    for line in lines:
        if (len(line) > 1 and 'POSTGRESQL' not in line and 'ORACLE' not in line):
            result = line.split('=')
            if ('HOST_POST' in result[0]):
                HOST_POST = result[1].replace('\n','')
            elif ('PORT_POST' in result[0]):
                PORT_POST = result[1].replace('\n','')
            elif ('DB_NAME_POST' in result[0]):
                DB_NAME_POST = result[1].replace('\n','')
            elif ('USER_POST' in result[0]):
                USER_POST = result[1].replace('\n','')
            elif ('PWD_POST' in result[0]):
                PWD_POST = result[1].replace('\n','')
            elif ('HOST_ORAC' in result[0]):
                HOST_ORAC = result[1].replace('\n','')
            elif ('PORT_ORAC' in result[0]):
                PORT_ORAC = result[1].replace('\n','')
            elif ('SERVICE_NAME' in result[0]):
                SERVICE_NAME = result[1].replace('\n','')
            elif ('USER_ORAC' in result[0]):
                USER_ORAC = result[1].replace('\n','')
            elif ('PWD_ORAC' in result[0]):
                PWD_ORAC = result[1].replace('\n','')

# Entry point
if __name__ == '__main__':
    getDBConnection()

    # Exit with an error if there are no arguments
    if (len(args) < 2):
        print ('>> Please check arguement. quitting.')
        sys.exit(9)
    else:
        GetProcdate(args)
```

I think there is a better way to write the process of reading the DBConnection file, but since I'm not very familiar with the Python-specific way of writing Getters/Setters, this is the result I wrote as code that I can understand. The fact that global variables cannot be used unless they are declared `global` in a method was quite new to me, as I was used to Java. On the other hand, I think it also means that global variables should not be used in different methods as much as possible.

Another attractive feature is that you can easily write files not only to text but also to binary data. In Java, it would have been quite complicated to write things like streams and buffers (and I've only worked with text, so I don't know if you can apply the same method to binary), but in Python you can easily do it without importing any modules. Of course, considering performance, it might have been better to write it in Java, but I think this kind of simplicity will definitely help improve productivity.

I also think that Python's Linux-friendly structure is attractive. You can easily check the exit code by setting `sys.exit()` to determine the result of the operation, and you can also accept arguments from the command line through `sys.args`. Since it is easy and Linux-friendly, I have come to think that it is better to use Python than shell scripts when script processing on Linux is required. There are also reports that it has better performance than the shell.

The end result was a post praising Python, but I think it's an attractive language for beginners to programming or developers who often write scripts from Linux servers, so I'd like everyone to use Python as well. Let's have fun and develop easily!

[^1]: Abbreviation for Simple Storage Service, and as the name suggests, it is a service that can be mounted on the OS and used like a normal disk.
