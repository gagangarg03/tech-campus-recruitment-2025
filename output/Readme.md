Output Folder Structure and Contents
The output/ folder contains the results of running the program. For each query, the program generates a file named output_YYYY-MM-DD.txt, where YYYY-MM-DD is the date provided as input. Each file contains all the log entries for the specified date.

Example Output Folder Structure:
output/
│
├── output_2024-12-01.txt
├── output_2024-12-02.txt
└── output_2024-12-03.txt

Example Output Files:
output_2024-12-01.txt:

2024-12-01 14:23:45 INFO User logged in  
2024-12-01 14:24:10 ERROR Failed to connect to the database  
2024-12-01 15:00:00 WARN Disk space running low  

output_2024-12-02.txt:

2024-12-02 09:15:30 WARN Disk space running low  
2024-12-02 10:00:00 INFO Backup completed successfully 
 
output_2024-12-03.txt:

(Empty file, no logs found for this date)



How to Generate Output Files
Run the Program:
Use the command:

bash
./extract_logs <YYYY-MM-DD> <approach> <num_threads>
Example:

bash
./extract_logs 2024-12-01 parallel 8
Check the output/ Folder:
After running the program, navigate to the output/ directory:

bash
cd output
ls
You should see a file named output_YYYY-MM-DD.txt.

View the Output File:
Use a text editor or command-line tool to view the contents of the output file:

bash
cat output_2024-12-01.txt
Notes
Ensure the output/ Directory Exists:
Before running the program, create the output/ directory:

bash
mkdir output
Empty Output:
If no logs are found for the specified date, the program will print:


No logs found for the specified date.
The output file will still be created but will be empty.

Multiple Queries:
If you run the program multiple times with different dates, each query will generate a separate output file in the output/ folder.

Example Directory Structure After Running the Program

project/
│
├── src/
│   └── extract_logs.cpp           # Source code
├── large_log_file.txt             # Large log file (1 TB)
├── output/                        # Output directory
│   ├── output_2024-12-01.txt      # Output file for 2024-12-01
│   ├── output_2024-12-02.txt      # Output file for 2024-12-02
│   └── output_2024-12-03.txt      # Output file for 2024-12-03
└── README.md                      # Documentation
