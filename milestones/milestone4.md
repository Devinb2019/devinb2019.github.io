# Milestone 4: Enhancement Three – Databases

## Post-Enhancement
```plaintext
#include <iostream>
#include <fstream>
#include <unordered_map>
#include <vector>
#include <string>
#include <algorithm>
#include <cctype>
#include <functional>
#include <limits>
#include <filesystem>
#include "sqlite3.h"

// -----------------------------------------------------------------------------
// Grocery Tracker Application — Capstone Project Enhancements
// Milestone Four (Databases)
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
//
// Category Three Enhancements (Databases):
//  • Integrated SQLite instead of flat-file storage
//  • Created table with item name (primary key) and frequency
//  • Loaded state from DB on startup and applied upsert logic on save
//  • Batched updates inside a transaction for consistency and performance
//  • Migrated legacy data from original input and frequency.dat into SQLite
//  • Added interactive add/update functionality allowing users to increment or set counts manually
//  • Added remove/decrement functionality with safe deletion when counts drop to zero
// -----------------------------------------------------------------------------

constexpr auto DB_FILENAME = "grocery_frequency.db";
constexpr auto LEGACY_TEXT_INPUT = "CS210_Project_Three_Input_File.txt";
constexpr auto LEGACY_FREQ_FILE = "frequency.dat";
constexpr auto MIGRATED_LEGACY_MARKER = "frequency.merged";

// Utility: normalize strings to lowercase so lookups are case-insensitive.
// This prevents duplicates like "Apple" vs "apple" and simplifies user queries.
static std::string toLower(const std::string& str) {
    std::string result = str;
    std::transform(result.begin(), result.end(), result.begin(),
        [](unsigned char ch) { return std::tolower(ch); });
    return result;
}

// -----------------------------------------------------------------------------
// FileHandler: encapsulates everything SQLite-related so that the rest of the
// application doesn't need to care about SQL syntax, transactions, or persistence.
// -----------------------------------------------------------------------------
class FileHandler {
public:
    // Open (or create) the database file and ensure the table schema exists.
    // Returns false if the database can't be opened or the table fails to be created.
    static bool initializeDatabase(sqlite3*& db) {
        int rc = sqlite3_open(DB_FILENAME, &db);
        if (rc != SQLITE_OK) {
            std::cerr << "Cannot open database: " << sqlite3_errmsg(db) << std::endl;
            return false;
        }

        // Ensure we have the expected table. If it doesn't exist, create it.
        const char* createTableSQL =
            "CREATE TABLE IF NOT EXISTS frequencies ("
            "item TEXT PRIMARY KEY,"
            "count INTEGER NOT NULL"
            ");";
        char* err = nullptr;
        rc = sqlite3_exec(db, createTableSQL, nullptr, nullptr, &err);
        if (rc != SQLITE_OK) {
            std::cerr << "Failed to create table: " << err << std::endl;
            sqlite3_free(err);
            return false;
        }
        return true;
    }

    // Load all existing item counts from SQLite into an in-memory map.
    // If any database preparation step fails, logs the error and returns what has
    // been collected so far (possibly empty).
    static std::unordered_map<std::string, int> loadData(sqlite3* db) {
        std::unordered_map<std::string, int> data;
        const char* query = "SELECT item, count FROM frequencies;";
        sqlite3_stmt* stmt = nullptr;

        if (sqlite3_prepare_v2(db, query, -1, &stmt, nullptr) != SQLITE_OK) {
            std::cerr << "Prepare failed: " << sqlite3_errmsg(db) << std::endl;
            return data;
        }

        // Walk through each row and populate the map.
        while (sqlite3_step(stmt) == SQLITE_ROW) {
            const unsigned char* itemText = sqlite3_column_text(stmt, 0);
            int count = sqlite3_column_int(stmt, 1);
            if (itemText) {
                std::string item(reinterpret_cast<const char*>(itemText));
                data[item] = count;
            }
        }

        sqlite3_finalize(stmt);
        return data;
    }

    // Persist the in-memory frequency map back to the database, using upsert logic.
    // Wraps all modifications in a transaction to ensure atomicity and better performance.
    static void saveData(sqlite3* db, const std::unordered_map<std::string, int>& data) {
        char* err = nullptr;
        if (sqlite3_exec(db, "BEGIN TRANSACTION;", nullptr, nullptr, &err) != SQLITE_OK) {
            std::cerr << "Failed to begin transaction: " << err << std::endl;
            sqlite3_free(err);
            return;
        }

        // Upsert: insert new rows, or update existing ones with the latest count.
        const char* upsertSQL =
            "INSERT INTO frequencies (item, count) VALUES (?, ?)\n"
            "ON CONFLICT(item) DO UPDATE SET count=excluded.count;";

        sqlite3_stmt* stmt = nullptr;
        if (sqlite3_prepare_v2(db, upsertSQL, -1, &stmt, nullptr) != SQLITE_OK) {
            std::cerr << "Prepare upsert failed: " << sqlite3_errmsg(db) << std::endl;
            return;
        }

        for (const auto& entry : data) {
            sqlite3_reset(stmt);              // reuse the prepared statement
            sqlite3_clear_bindings(stmt);     // clear previous bindings
            sqlite3_bind_text(stmt, 1, entry.first.c_str(), -1, SQLITE_TRANSIENT); // item
            sqlite3_bind_int(stmt, 2, entry.second); // count
            if (sqlite3_step(stmt) != SQLITE_DONE) {
                std::cerr << "Upsert failed for " << entry.first << ": " << sqlite3_errmsg(db) << std::endl;
            }
        }

        sqlite3_finalize(stmt);

        if (sqlite3_exec(db, "COMMIT;", nullptr, nullptr, &err) != SQLITE_OK) {
            std::cerr << "Failed to commit transaction: " << err << std::endl;
            sqlite3_free(err);
        }
    }

    // Cleanly close the SQLite connection if it was opened.
    static void closeDatabase(sqlite3* db) {
        if (db) sqlite3_close(db);
    }
};

// -----------------------------------------------------------------------------
// GroceryTracker: core application logic for manipulating the frequency data.
// All operations are done against an in-memory unordered_map; persistence is
// handled separately via FileHandler.
// -----------------------------------------------------------------------------
class GroceryTracker {
private:
    std::unordered_map<std::string, int> itemFrequency;

    // Internal helper to avoid duplicating iteration logic for different views.
    void forEachFrequency(const std::function<void(const std::string&, int)>& action) const {
        for (const auto& entry : itemFrequency) {
            action(entry.first, entry.second);
        }
    }

public:
    // Replace the internal state wholesale (used after loading from DB).
    void setFrequencyMap(const std::unordered_map<std::string, int>& data) {
        itemFrequency = data;
    }

    // Merge another map into ours, summing counts when keys collide. Used for
    // migrating legacy data without losing what's already persisted.
    void mergeFrequencyMap(const std::unordered_map<std::string, int>& other) {
        for (const auto& entry : other) {
            itemFrequency[entry.first] += entry.second;
        }
    }

    // Search for a specific item in a case-insensitive way.
    void searchItem() const {
        std::cout << "Enter item to search: ";
        std::string query;
        std::getline(std::cin, query);
        if (query.empty()) {
            std::cout << "No input entered.\n";
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

    // Add a new item or update an existing one. User can choose to increment by
    // one (blank input) or set an explicit count. Defensive parsing avoids crashes.
    void addOrUpdateItem() {
        std::cout << "Enter item to add/update: ";
        std::string input;
        std::getline(std::cin, input);
        if (input.empty()) {
            std::cout << "No item entered.\n";
            return;
        }
        std::string key = toLower(input);

        std::cout << "Enter count to set (or leave blank to increment by 1): ";
        std::string countStr;
        std::getline(std::cin, countStr);
        if (countStr.empty()) {
            // default behavior: bump existing count or create with 1
            itemFrequency[key] += 1;
        }
        else {
            try {
                int val = std::stoi(countStr);
                if (val < 0) {
                    std::cout << "Negative counts not allowed.\n";
                    return;
                }
                itemFrequency[key] = val;
            }
            catch (...) {
                std::cout << "Invalid number; skipping update.\n";
                return;
            }
        }
        std::cout << "Item '" << key << "' is now " << itemFrequency[key] << " time(s).\n";
    }

    // Decrement or remove an item. Allows full deletion with "all" or partial
    // decrement; deletion occurs if requested decrement is >= current count.
    void removeItem() {
        std::cout << "Enter item to remove/decrement: ";
        std::string input;
        std::getline(std::cin, input);
        if (input.empty()) {
            std::cout << "No input entered.\n";
            return;
        }
        std::string key = toLower(input);
        auto it = itemFrequency.find(key);
        if (it == itemFrequency.end()) {
            std::cout << "Item '" << key << "' not found.\n";
            return;
        }

        std::cout << "Current count is " << it->second << ". Enter amount to decrement (or 'all' to delete): ";
        std::string amt;
        std::getline(std::cin, amt);
        if (amt == "all") {
            itemFrequency.erase(key);
            std::cout << "Item '" << key << "' deleted.\n";
            return;
        }
        try {
            int dec = std::stoi(amt);
            if (dec <= 0) {
                std::cout << "Must decrement by positive amount.\n";
                return;
            }
            if (dec >= it->second) {
                itemFrequency.erase(key);
                std::cout << "Item '" << key << "' count dropped to zero and was removed.\n";
            }
            else {
                it->second -= dec;
                std::cout << "Item '" << key << "' decremented to " << it->second << " time(s).\n";
            }
        }
        catch (...) {
            std::cout << "Invalid number; no change made.\n";
        }
    }

    // Print all items with their counts in no particular order.
    void printFrequencyList() const {
        forEachFrequency([](const std::string& item, int count) {
            std::cout << item << ": " << count << '\n';
            });
    }

    // Show a simple ASCII histogram visualization of counts.
    void printHistogram() const {
        forEachFrequency([](const std::string& item, int count) {
            std::cout << item << " ";
            for (int i = 0; i < count; ++i) std::cout << '*';
            std::cout << '\n';
            });
    }

    // Provide a sorted view (descending) of items by frequency. Useful for
    // quickly identifying the most common grocery items.
    void printSortedFrequencyList() const {
        std::vector<std::pair<std::string, int>> entries(itemFrequency.begin(), itemFrequency.end());
        std::sort(entries.begin(), entries.end(), [](auto& a, auto& b) { return a.second > b.second; });
        std::cout << "\n-- Items sorted by frequency --\n";
        for (const auto& entry : entries) {
            std::cout << entry.first << ": " << entry.second << '\n';
        }
    }

    // External access used when saving to the database.
    const std::unordered_map<std::string, int>& getFrequencyMap() const {
        return itemFrequency;
    }

    // Legacy migration: read the original plain-text input file and merge its
    // contents in (case-insensitive). This only adds; it doesn't overwrite.
    void ingestFromTextFile(const std::string& filename) {
        std::ifstream inputFile(filename);
        if (!inputFile.is_open()) {
            std::cerr << "Legacy text file not present: " << filename << '\n';
            return;
        }
        std::string item;
        while (inputFile >> item) {
            const std::string key = toLower(item);
            itemFrequency[key]++;
        }
        inputFile.close();
    }

    // Legacy migration: read the old frequency.dat flat file (item count pairs)
    // and return a map so caller can merge it explicitly.
    std::unordered_map<std::string, int> loadLegacyFrequencyFile(const std::string& filename) const {
        std::unordered_map<std::string, int> legacy;
        std::ifstream in(filename);
        if (!in.is_open()) return legacy;

        std::string item;
        int count;
        while (in >> item >> count) {
            std::string key = toLower(item);
            legacy[key] += count;
        }
        return legacy;
    }
};

// -----------------------------------------------------------------------------
// MenuManager: isolates all the user interaction logic so the core tracker stays
// testable and separate from I/O concerns.
// -----------------------------------------------------------------------------
class MenuManager {
public:
    static void displayMenu() {
        std::cout << "\n--- Main Menu ---\n"
            << "1) Search for an item\n"
            << "2) Show all frequencies\n"
            << "3) Display histogram\n"
            << "4) Show sorted frequencies\n"
            << "5) Add or set an item\n"
            << "6) Remove/decrement an item\n"
            << "7) Quit\n"
            << "Enter choice (1-7): ";
    }

    // Dispatches the user's numeric selection to the appropriate tracker action.
    static void handleUserSelection(int choice, GroceryTracker& tracker) {
        switch (choice) {
        case 1: tracker.searchItem(); break;
        case 2: tracker.printFrequencyList(); break;
        case 3: tracker.printHistogram(); break;
        case 4: tracker.printSortedFrequencyList(); break;
        case 5: tracker.addOrUpdateItem(); break;
        case 6: tracker.removeItem(); break;
        case 7: std::cout << "Quitting and saving data...\n"; break;
        default: std::cout << "Invalid choice; please select 1-7.\n";
        }
    }

    // Main interaction loop. Validates input to avoid crashes and ensures the
    // newline is consumed so subsequent getline calls behave correctly.
    static void runMenu(GroceryTracker& tracker) {
        int choice = 0;
        while (choice != 7) {
            displayMenu();
            if (!(std::cin >> choice)) {
                std::cin.clear(); // reset failbit
                std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n'); // drop bad input
                std::cout << "Please enter a valid number.\n";
                continue;
            }
            std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n'); // consume leftover newline
            handleUserSelection(choice, tracker);
        }
    }
};

// -----------------------------------------------------------------------------
// main: orchestrates database initialization, migrating legacy data, user
// interaction, and final persistence. SQLite is the single authoritative store
// after this bootstrapped merge.
// -----------------------------------------------------------------------------
int main() {
    sqlite3* db = nullptr;
    if (!FileHandler::initializeDatabase(db)) {
        return 1; // cannot proceed without a valid database
    }

    GroceryTracker tracker;

    // 1. Load whatever is already in the database.
    auto existing = FileHandler::loadData(db);
    tracker.setFrequencyMap(existing);

    // 2. Merge in the original text input if present. This ensures the base
    //    dataset from earlier versions is preserved.
    tracker.ingestFromTextFile(LEGACY_TEXT_INPUT);

    // 3. Merge in the older flat-file frequency.dat contents if it exists, then
    //    rename it so we don't re-import on subsequent runs.
    if (std::filesystem::exists(LEGACY_FREQ_FILE)) {
        auto legacyFreq = tracker.loadLegacyFrequencyFile(LEGACY_FREQ_FILE);
        tracker.mergeFrequencyMap(legacyFreq);

        std::error_code ec;
        std::filesystem::rename(LEGACY_FREQ_FILE, MIGRATED_LEGACY_MARKER, ec);
        if (ec) {
            std::cerr << "Warning: unable to rename legacy frequency file: " << ec.message() << '\n';
        }
    }

    // 4. Persist the merged state back into SQLite.
    FileHandler::saveData(db, tracker.getFrequencyMap());

    // 5. Let the user interact with the merged dataset.
    MenuManager::runMenu(tracker);

    // 6. Save any updates the user made during the session.
    FileHandler::saveData(db, tracker.getFrequencyMap());

    FileHandler::closeDatabase(db);
    return 0;
}
