# Milestone 3: Enhancement Two – Algorithms and Data Structure

## Post-Enhancement
```cpp
#include <iostream>
#include <fstream>
#include <unordered_map>
#include <vector>
#include <string>
#include <algorithm>
#include <cctype>
#include <functional>

// -----------------------------------------------------------------------------
// Grocery Tracker Application — Capstone Project Enhancements
// Milestone Three
// 
// Category One Enhancements (Software Design & Engineering):
//  • Single-responsibility classes: FileHandler, GroceryTracker, MenuManager
//  • Defensive input validation to prevent bad input and infinite loops
//  • Named constants for file names instead of hard-coded strings
//  • Case-insensitive item keys via toLower helper
//  • Consolidated map iteration with forEachFrequency helper
// 
// Category Two Enhancements (Algorithms & Data Structures):
//  • Replaced ordered map with std::unordered_map for O(1) average lookups/inserts
//  • Added 'sorted-frequency' view by copying into vector and sorting by count
//  • Introduced a new menu option to display the sorted-frequency list directly
// -----------------------------------------------------------------------------

constexpr auto INPUT_FILENAME = "CS210_Project_Three_Input_File.txt";
constexpr auto OUTPUT_FILENAME = "frequency.dat";

// Utility function: convert string to lowercase for consistent keying
static std::string toLower(const std::string& str) {
    std::string result = str;
    std::transform(result.begin(), result.end(), result.begin(),
        [](unsigned char ch) { return std::tolower(ch); });
    return result;
}

// -----------------------------------------------------------------------------
// FileHandler: handles loading and saving data, using unordered_map for speed
// -----------------------------------------------------------------------------
class FileHandler {
public:
    static std::unordered_map<std::string, int> loadData(const std::string& filename) {
        std::unordered_map<std::string, int> data;
        std::ifstream inputFile(filename);
        if (!inputFile.is_open()) {
            std::cerr << "Error opening file: " << filename << std::endl;
            return data;
        }
        std::string item;
        while (inputFile >> item) {
            // Normalize to lowercase to treat "Apple" and "apple" the same
            const std::string key = toLower(item);
            ++data[key];
        }
        inputFile.close();
        return data;
    }

    static void saveData(const std::string& filename,
        const std::unordered_map<std::string, int>& data) {
        std::ofstream outputFile(filename);
        if (!outputFile.is_open()) {
            std::cerr << "Error writing to file: " << filename << std::endl;
            return;
        }
        for (const auto& entry : data) {
            outputFile << entry.first << " " << entry.second << '\n';
        }
        outputFile.close();
    }
};

// -----------------------------------------------------------------------------
// GroceryTracker: core logic separated from I/O and UI
// -----------------------------------------------------------------------------
class GroceryTracker {
private:
    std::unordered_map<std::string, int> itemFrequency;

    // Helper: run a given action on each key/count pair
    void forEachFrequency(const std::function<void(const std::string&, int)>& action) const {
        for (const auto& entry : itemFrequency) {
            action(entry.first, entry.second);
        }
    }

public:
    // Inject data loaded by FileHandler
    void setFrequencyMap(const std::unordered_map<std::string, int>& data) {
        itemFrequency = data;
    }

    // Case-insensitive search with input validation
    void searchItem() const {
        std::cout << "Enter item to search: ";
        std::string query;
        std::cin >> query;
        if (std::cin.fail()) {
            std::cin.clear();
            std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
            std::cout << "Invalid input. Please enter a valid item.\n";
            return;
        }
        const auto key = toLower(query);
        auto it = itemFrequency.find(key);
        if (it != itemFrequency.end()) {
            std::cout << "Found '" << key << "' " << it->second << " time(s).\n";
        }
        else {
            std::cout << "Sorry, '" << key << "' wasn't found.\n";
        }
    }

    // Print each item and its count in arbitrary order
    void printFrequencyList() const {
        forEachFrequency([](const std::string& item, int count) {
            std::cout << item << ": " << count << '\n';
            });
    }

    // ASCII histogram view
    void printHistogram() const {
        forEachFrequency([](const std::string& item, int count) {
            std::cout << item << " ";
            for (int i = 0; i < count; ++i) std::cout << '*';
            std::cout << '\n';
            });
    }

    // New: sorted list by frequency descending
    void printSortedFrequencyList() const {
        std::vector<std::pair<std::string, int>> entries(
            itemFrequency.begin(), itemFrequency.end());
        std::sort(entries.begin(), entries.end(),
            [](auto& a, auto& b) { return a.second > b.second; });
        std::cout << "\n-- Items sorted by frequency --\n";
        for (const auto& entry : entries) {
            std::cout << entry.first << ": " << entry.second << '\n';
        }
    }

    // Expose raw data for saving
    const std::unordered_map<std::string, int>& getFrequencyMap() const {
        return itemFrequency;
    }
};

// -----------------------------------------------------------------------------
// MenuManager: UI loop separated from core logic; supports both new and old views
// -----------------------------------------------------------------------------
class MenuManager {
public:
    static void displayMenu() {
        std::cout << "\n--- Main Menu ---\n"
            << "1) Search for an item\n"
            << "2) Show all frequencies\n"
            << "3) Display histogram\n"
            << "4) Show sorted frequencies\n"
            << "5) Quit\n"
            << "Enter choice (1-5): ";
    }

    static void handleUserSelection(int choice, GroceryTracker& tracker) {
        switch (choice) {
        case 1: tracker.searchItem(); break;
        case 2: tracker.printFrequencyList(); break;
        case 3: tracker.printHistogram(); break;
        case 4: tracker.printSortedFrequencyList(); break;
        case 5: std::cout << "Quitting and saving data...\n"; break;
        default: std::cout << "Invalid choice; please select 1-5.\n";
        }
    }

    static void runMenu(GroceryTracker& tracker) {
        int choice = 0;
        while (choice != 5) {
            displayMenu();
            std::cin >> choice;
            if (std::cin.fail()) {
                std::cin.clear();
                std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
                std::cout << "Please enter a valid number.\n";
                continue;
            }
            handleUserSelection(choice, tracker);
        }
    }
};

// -----------------------------------------------------------------------------
// main: bring everything together—load, interact, save
// -----------------------------------------------------------------------------
int main() {
    auto data = FileHandler::loadData(INPUT_FILENAME);
    GroceryTracker tracker;
    tracker.setFrequencyMap(data);

    MenuManager::runMenu(tracker);

    FileHandler::saveData(OUTPUT_FILENAME, tracker.getFrequencyMap());
    return 0;
}
```
