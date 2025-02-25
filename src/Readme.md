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

// Naive approach: Full file scan
void naiveApproach(const string& logFilePath, const string& date, const string& outputFilePath) {
    ifstream logFile(logFilePath);
    ofstream outputFile(outputFilePath);

    string line;
    while (getline(logFile, line)) {
        if (line.substr(0, 10) == date) {
            outputFile << line << endl;
        }
    }

    logFile.close();
    outputFile.close();
}

// Binary search approach (assumes file is sorted by date)
void binarySearchApproach(const string& logFilePath, const string& date, const string& outputFilePath) {
    ifstream logFile(logFilePath, ios::binary | ios::ate);
    long fileSize = logFile.tellg();
    long low = 0, high = fileSize;
    string line;

    // Binary search to find the start offset
    while (low < high) {
        long mid = low + (high - low) / 2;
        logFile.seekg(mid);
        getline(logFile, line); // Move to the start of the next line
        string currentDate = line.substr(0, 10);

        if (currentDate < date) {
            low = mid + 1;
        } else {
            high = mid;
        }
    }

    long startOffset = low;

    // Binary search to find the end offset
    high = fileSize;
    while (low < high) {
        long mid = low + (high - low) / 2;
        logFile.seekg(mid);
        getline(logFile, line); // Move to the start of the next line
        string currentDate = line.substr(0, 10);

        if (currentDate <= date) {
            low = mid + 1;
        } else {
            high = mid;
        }
    }

    long endOffset = low;

    // Retrieve logs for the specified date
    logFile.seekg(startOffset);
    ofstream outputFile(outputFilePath);

    while (getline(logFile, line) && logFile.tellg() <= endOffset) {
        if (line.substr(0, 10) == date) {
            outputFile << line << endl;
        }
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
