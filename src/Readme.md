# Source Directory

Make sure that your source code is in the `src` directory.


# SOLUTION FOR THE PROBLEM STATEMENT

#include <iostream>
#include <fstream>
#include <unordered_map>
#include <string>
#include <sstream>
#include <vector>
#include <thread>
#include <mutex>
#include <algorithm>

using namespace std;

// Structure to store start and end offsets for a date
struct DateRange {
    long start;
    long end;
};

// Global index and mutex for thread safety
unordered_map<string, DateRange> index;
mutex indexMutex;

// Function to process a chunk of the file
void processChunk(const string& logFilePath, long startOffset, long endOffset) {
    ifstream logFile(logFilePath, ios::binary);
    logFile.seekg(startOffset);

    string line;
    long currentOffset = startOffset;
    string currentDate;

    while (getline(logFile, line) && currentOffset <= endOffset) {
        string date = line.substr(0, 10); // Extract YYYY-MM-DD from timestamp
        if (date != currentDate) {
            if (!currentDate.empty()) {
                lock_guard<mutex> lock(indexMutex);
                index[currentDate].end = currentOffset - 1;
            }
            currentDate = date;
            lock_guard<mutex> lock(indexMutex);
            index[currentDate].start = currentOffset;
        }
        currentOffset = logFile.tellg();
    }

    logFile.close();
}

// Function to build the index in parallel
void buildIndexParallel(const string& logFilePath, int numThreads) {
    ifstream logFile(logFilePath, ios::binary | ios::ate);
    long fileSize = logFile.tellg();
    logFile.close();

    vector<thread> threads;
    long chunkSize = fileSize / numThreads;

    for (int i = 0; i < numThreads; ++i) {
        long startOffset = i * chunkSize;
        long endOffset = (i == numThreads - 1) ? fileSize : (i + 1) * chunkSize;
        threads.emplace_back(processChunk, logFilePath, startOffset, endOffset);
    }

    for (auto& t : threads) {
        t.join();
    }
}

// Function to retrieve logs for a specific date
void retrieveLogs(const string& logFilePath, const string& date, const string& outputFilePath) {
    if (index.find(date) == index.end()) {
        cout << "No logs found for the specified date." << endl;
        return;
    }

    ifstream logFile(logFilePath, ios::binary);
    ofstream outputFile(outputFilePath);

    DateRange range = index.at(date);
    logFile.seekg(range.start);

    string line;
    while (getline(logFile, line) && logFile.tellg() <= range.end) {
        outputFile << line << endl;
    }

    logFile.close();
    outputFile.close();
}

int main(int argc, char* argv[]) {
    if (argc != 4) {
        cerr << "Usage: " << argv[0] << " <YYYY-MM-DD> <approach> <num_threads>" << endl;
        cerr << "Approaches: naive, indexing, parallel, binary" << endl;
        return 1;
    }

    string date = argv[1];
    string approach = argv[2];
    int numThreads = stoi(argv[3]);
    string logFilePath = "large_log_file.txt"; // Path to the large log file
    string outputFilePath = "output/output_" + date + ".txt";

    if (approach == "naive") {
        // Naive approach
        naiveApproach(logFilePath, date, outputFilePath);
    } else if (approach == "indexing") {
        // Indexing approach
        buildIndexParallel(logFilePath, numThreads);
        retrieveLogs(logFilePath, date, outputFilePath);
    } else if (approach == "parallel") {
        // Parallel indexing approach
        buildIndexParallel(logFilePath, numThreads);
        retrieveLogs(logFilePath, date, outputFilePath);
    } else if (approach == "binary") {
        // Binary search approach
        binarySearchApproach(logFilePath, date, outputFilePath);
    } else {
        cerr << "Invalid approach. Use naive, indexing, parallel, or binary." << endl;
        return 1;
    }

    cout << "Logs for " << date << " have been saved to " << outputFilePath << endl;
    return 0;
}



EXPLAINATION FOR THE APPROACH :

THERE ARE SO MANY APPROACHES TO SOLVE THIS PROBLEM LET'S DISCUSS IN DETAIL -

1. Naive Approach (Full File Scan):
Description: Read the entire file line by line and filter logs for the specified date.

Pros:

Simple to implement.

No preprocessing required.

Cons:

Extremely slow for large files (1 TB). Time complexity is O(n), where n is the file size.

Not feasible for real-world use due to high time and resource consumption.

2. Date-Based Indexing:
Description: Preprocess the file to create an index that maps each date to its start and end byte offsets in the file.

Pros:

Fast retrieval for specific dates. Time complexity for querying is O(1) after indexing.

Efficient memory usage as only the index is stored in memory.

Cons:

Requires one-time preprocessing to build the index.

Indexing time is O(n), but it is a one-time cost.

3. Parallel Indexing:
Description: Use multi-threading to build the index in parallel, dividing the file into chunks and processing each chunk in a separate thread.

Pros:

Faster indexing compared to the sequential approach.

Scalable with the number of CPU cores.

Cons:

Requires careful handling of thread synchronization.

Slightly more complex implementation.

4. Binary Search:
Description: Assume the file is sorted by date and use binary search to locate the start and end offsets for the specified date.

Pros:

Efficient for sorted files. Time complexity is O(log n) for finding the date range.

No preprocessing required if the file is already sorted.

Cons:

Requires the file to be sorted by date.

Sorting a 1 TB file is computationally expensive.

5. Database Storage:
Description: Store the logs in a database (e.g., SQLite, MySQL) and query using SQL.

Pros:

Easy to query and manage.

Supports complex queries (e.g., range queries, filtering by log level).

Cons:

Importing 1 TB of data into a database is time-consuming and resource-intensive.

Requires additional infrastructure (e.g., database server).



CHOOSEN APPROACH: PARALLEL INDEXING 

Features of the Final Solution:
Indexing:

The log file is preprocessed to create an index that maps each date to its start and end byte offsets.

Indexing is done in parallel using multi-threading to reduce preprocessing time.

Querying:

The index is used to locate the byte range for the specified date.

Only the relevant portion of the file is read, minimizing I/O operations.

Output:

The retrieved logs are saved to an output file named output/output_YYYY-MM-DD.txt.

Scalability:

The number of threads can be adjusted based on the available CPU cores, making the solution scalable.



Comparison of Approaches :
Approach	    Preprocessing Time	 Query Time	   Memory Usage	   Implementation Complexity
Naive	  		  None                   O(n)         Low	              Low
Indexing	    O(n)                  O(1)		      Moderate          Moderate
Parallel Indexing	O(n/p)	           O(1)	        Moderate	         High
Binary Search	None (if sorted)	   O(log n)	        Low	           Moderate
Database Storage	High	           O(log n)	        High	          High


