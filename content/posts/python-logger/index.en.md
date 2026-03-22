---
title: "I want to log with Python"
date: 2019-06-15
translationKey: "posts/python-logger"
categories: 
  - python
image: "../../images/python.webp"
tags:
  - log
  - python
---
When it comes to coding, I think that if there is one thing that is extremely important when it comes to putting together logic, it is to check whether the code you are writing is correct. First, we write specifications and then write code while checking whether the requirements are met. However, you cannot be sure that the code will work as expected unless you try it. To determine whether the behavior is correct, check whether the variables and objects handled in the code contain the correct data. IDEs such as Eclipse have debugging features that allow you to set checkpoints and monitor the flow of your code, but there are times when this is not possible. There is also a method of using standard output, but I am worried because I cannot track the record after execution.

One way to do this is to use custom logs. By creating a logger from the beginning that outputs proper logs, you can determine whether the program you created is working as designed. And even after the program is completed, it can be used effectively to check for network failures and data problems. However, writing a logger that emits custom logs seems difficult. What should I do when I first receive instructions to generate logs? Is there no standard model? I was worried. However, it didn't seem that difficult; all I had to do was decide on branches (messages, codes, etc.) depending on the situation, and output them as a text file.

Now, let's show you how we created the actual logger through code.

## Create a logger

The structure of the logger that I want to create this time is simple. When called by the main program, it receives a log code (classification number, etc.) and job code (work processing number, etc.) as arguments, and determines the message to be output from the log code. Although the job code is not required, I would like to make the log code mandatory. Also, since we want to know when the log was written, we also include the date and time in the message. The format of the log output from this requirement is as follows.

> 2019/06/15 12:00:00 LOG0001: Job code 1234 ended normally.I also want to put `---- Job Log ----` at the top of the log file to indicate that this is a log file. In this way, we will determine the basic requirements and implement them. A simplified version of the code I implemented is below.

```python
# Import os to check whether the log file exists, and datetime to add timestamps
import os
import datetime

# Logging function (job_code may be omitted, so it has a default value)
def writeLog(log_code, job_code=''):
    # Specify the log file path (absolute path)
    log_path = '/log/program_log.log'

    # Decide the message based on the log code
    if (log_code == 'LOGI001'):
        log_message = 'Running job code ' + job_code + '.'
    elif (log_code == 'LOGI002'):
        log_message = 'Job code ' + job_code + ' completed successfully.'
    elif (log_code == 'LOGW001'):
        log_message = 'No job code was provided. The program failed to start.'
    elif (log_code == 'LOGE001'):
        log_message = 'Job code ' + job_code + ' ended abnormally.'
    else:
        log_message = 'An error occurred that could not be identified.'

    # Create a log file with a header if it does not already exist
    if (not os.path.exists(log_path)):
        with open(log_path, mode='a') as log:
            log.write('----- Job Log -----\n')
    else 
    # Append to the log file if it already exists
        with open(log_path, mode='a') as log:
            log.write(datetime.datetime.today().strftime('%Y/%m/%d %H:%M:%S') + ' ' + log_code + ': ' + log_message + '\n')
```

Now, just call the logger in the main program and pass the log code or job code at that time, and the log will be left. If you want to reduce or increase the types of logs, you can do it simply by modifying the if statement.

## Call the logger in the main program

In the main program, first decide when to write the log (call the logger). For example, this time I had a task to connect to a DB, so I wrote a logger if the DB connection failed. In addition to that, I would also like to keep logs regarding the status at the start and end of the process.

The code for the main program, also simplified, is as follows. Just import the logger you created earlier and call it when you want to output logs. However, if the location of the logger file is in a different path from the main program being executed, the import format will change, so be careful. The implementation here assumes that the main program and logger are executed from the same path. Please take a look at the code below.

```python
# Simplified main program
# Import sys for exit codes and command-line arguments, and the logger's writeLog function for logging
import sys
from logger import writeLog

# Variable for command-line arguments at startup
args = sys.argv

# Example function
def program(job_code):
    # Write a log at the start of processing
    writeLog('LOGI001', job_code)
    # Normal processing flow
    try:
        print('Perform some processing')
        # Write a log for successful completion
        writeLog('LOGI002')
        # Return code is 0 on success
        sys.exit(0)
    # Handle failures during processing
    except:
        # Write a log for abnormal termination
        writeLog('LOGE001')
        # Return code is 9 on failure
        sys.exit(9)

# Program entry point
if __name__ == '__main__':
    # Check whether an argument (job code) was provided
    if (args[1] in None):
        # Write a warning log
        writeLog('LOGW001')
        # Return code is 1 on warning
        sys.exit(1)
    else:
        program(args[1])
```

## Finally

Decide where to call the logger depending on what kind of log you want to output. By passing arguments, you can decide what logs to output in which cases. Although it is not a difficult task to implement, by separating the logger in this way and calling it only when necessary, it is an efficient structure in that no matter how long the main program code becomes, you only need to call the function. In the first place, functions are written to reduce repetition.

I did a little more research and found that Python already provides logger as a module. It seems that you can specify the log level and output a string as standard output. I think it would be a good way to apply this and write code to draw in a file.

While writing code at work, I started to think that what's important is not what commands and function names I remember, but what structure I use to implement them. These days, it's easy to get courses and tips on basic grammar in books and on the internet, but I think this type of design knowledge is something that can only be obtained through practical experience. I'm looking forward to seeing what kind of code I'll write in the future.
