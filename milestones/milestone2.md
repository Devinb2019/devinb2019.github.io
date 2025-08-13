# Milestone 2: Enhancement One – Software Design and Engineering

## Post-Enhancement
```plaintext
#include <iostream>
#include <fstream>
#include <map>
#include <string>
#include <algorithm>
#include <cctype>
#include <functional>

// -----------------------------------------------------------------------------
// Grocery Tracker Application Milestone Two Enhancement One: Software Designand Engineering
// This console program reads grocery item names from a text file, counts how many
// times each item appears (case-insensitive), lets the user search for specific
// items, displays a full list or histogram of frequencies, and then saves the
// results back to disk before exiting.
// -----------------------------------------------------------------------------

// Instead of sprinkling literal strings throughout the code, we define these
// constants to make updates easier down the road.
constexpr auto INPUT_FILENAME = "CS210_Project_Three_Input_File.txt";  // Source data
constexpr auto OUTPUT_FILENAME = "frequency.dat";                      // Where tallies are saved

// A simple utility: transform any string into its lowercase equivalent.
// This ensures "Apple", "apple", and "APPLE" all map to the same key.
static std::string toLower(const std::string& str) {
    std::string result = str;
    std::transform(result.begin(), result.end(), result.begin(),
        [](unsigned char ch) { return std::tolower(ch); });
    return result;
}

// -----------------------------------------------------------------------------
// FileHandler: takes care of everything related to reading and writing files.
// Keeping file I/O separate helps keep our core logic clean and focused.
// -----------------------------------------------------------------------------
class FileHandler {
public:
    // loadData: opens the given file, reads each word as an item name,
    // normalizes it, and increments its count in a map. If opening fails,
    // reports an error and returns an empty map.
    static std::map<std::string, int> loadData(const std::string& filename) {
        std::map<std::string, int> data;
        std::ifstream inputFile(filename);
        if (!inputFile.is_open()) {
            std::cerr << "Error opening file: " << filename << std::endl;
            return data;
        }

        std::string item;
        // Read until EOF; each space- or newline-delimited token is one item.
        while (inputFile >> item) {
            const std::string key = toLower(item);
            ++data[key];  // Count it
        }

        inputFile.close();  // Always close streams when done
        return data;
    }

    // saveData: writes each entry of the frequency map back to disk.
    // Format: <item> <count> per line. If writing fails, an error is shown.
    static void saveData(const std::string& filename,
        const std::map<std::string, int>& data) {
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
// GroceryTracker: holds the map of items to counts and provides methods to
// interact with that data, like searching, listing, or drawing histograms.
// -----------------------------------------------------------------------------
class GroceryTracker {
private:
    std::map<std::string, int> itemFrequency;

    // A helper to run the same loop code for both the list and histogram views.
    void forEachFrequency(const std::function<void(const std::string&, int)>& action) const {
        for (const auto& entry : itemFrequency) {
            action(entry.first, entry.second);
        }
    }

public:
    // Initialize our internal map with data already loaded from FileHandler.
    void setFrequencyMap(const std::map<std::string, int>& data) {
        itemFrequency = data;
    }

    // Prompt the user, read their query, normalize it, and report the count.
    // Includes basic validation so the program won't crash on bad input.
    void searchItem() const {
        std::cout << "Enter item to search: ";
        std::string query;
        std::cin >> query;

        if (std::cin.fail()) {
            // If extraction fails, clear the error and skip the rest of the line
            std::cin.clear();
            std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
            std::cout << "Oops, that didn’t look like a valid item. Try again.\n";
            return;
        }

        const auto key = toLower(query);
        auto it = itemFrequency.find(key);
        if (it != itemFrequency.end()) {
            std::cout << "Found '" << key << "' " << it->second << " time(s).\n";
        }
        else {
            std::cout << "Sorry, '" << key << "' wasn't in the list.\n";
        }
    }

    // Print every item with its count, one per line.
    void printFrequencyList() const {
        forEachFrequency([](const std::string& item, int count) {
            std::cout << item << ": " << count << '\n';
            });
    }

    // Display a simple ASCII histogram: item name followed by '*' repeated count times.
    void printHistogram() const {
        forEachFrequency([](const std::string& item, int count) {
            std::cout << item << " ";
            for (int i = 0; i < count; ++i) std::cout << '*';
            std::cout << '\n';
            });
    }

    // Provide external access to the raw data for saving, testing, etc.
    const std::map<std::string, int>& getFrequencyMap() const {
        return itemFrequency;
    }
};

// -----------------------------------------------------------------------------
// MenuManager: handles all the user interface loops and choices, so that our
// core logic stays uncluttered by UI code. We separate display, input, and
// choice handling for clarity.
// -----------------------------------------------------------------------------
class MenuManager {
public:
    // Show the list of actions the user can take.
    static void displayMenu() {
        std::cout << "\n--- Main Menu ---\n"
            << "1) Search for a specific item\n"
            << "2) Show all item frequencies\n"
            << "3) Display histogram view\n"
            << "4) Quit\n"
            << "Please enter a number (1-4): ";
    }

    // Given a numeric choice, invoke the corresponding GroceryTracker behavior.
    // If the number is out of range, gently remind the user of valid options.
    static void handleUserSelection(int choice, GroceryTracker& tracker) {
        switch (choice) {
        case 1:
            tracker.searchItem();        // Look up a single item
            break;
        case 2:
            tracker.printFrequencyList(); // Full list
            break;
        case 3:
            tracker.printHistogram();    // Graphical stars
            break;
        case 4:
            std::cout << "Goodbye! Saving data now...\n";
            break;
        default:
            std::cout << "That’s not a valid choice. Please pick 1-4.\n";
        }
    }

    // Continuously loop until the user opts to exit. Validate inputs
    // to prevent crashes and guide the user back on track if necessary.
    static void runMenu(GroceryTracker& tracker) {
        int choice = 0;
        while (choice != 4) {
            displayMenu();
            std::cin >> choice;

            if (std::cin.fail()) {
                // Clear the bad state, remove leftover input, and retry
                std::cin.clear();
                std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
                std::cout << "Please enter a valid number, not letters or symbols.\n";
                continue;
            }

            handleUserSelection(choice, tracker);
        }
    }
};

// -----------------------------------------------------------------------------
// main: the entry point. We load existing data, hand control to the menu,
// then save any updates before the program ends gracefully.
// -----------------------------------------------------------------------------
int main() {
    // Step 1: Bring data into memory
    auto data = FileHandler::loadData(INPUT_FILENAME);
    GroceryTracker tracker;
    tracker.setFrequencyMap(data);

    // Step 2: Let the user inspect or modify the data
    MenuManager::runMenu(tracker);

    // Step 3: Write the final counts back to disk
    FileHandler::saveData(OUTPUT_FILENAME, tracker.getFrequencyMap());
    return 0;
}


