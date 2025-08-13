# Pre-Enhancement Code (Shared for All Milestones)

```plaintext
// #include <iostream>
#include <fstream>
#include <map>

class GroceryTracker {
private:
    std::map<std::string, int> itemFrequency; // Map to store item frequencies

public:
    // Function to read the input file
    void processFile(const std::string& filename);

    // Function to search for the frequency
    void searchItem();

    // Function to print the frequency list of all the items
    void printFrequencyList();

    // Function to print a histogram representation all the item frequencies
    void printHistogram();

    // Function to save data to the "frequency.dat" file
    void saveDataToFile();

    // Function to run the main menu for user interaction
    void runMenu();
};

void GroceryTracker::processFile(const std::string& filename) {
    // Open the input file
    std::ifstream inputFile(filename);
    if (!inputFile.is_open()) {
        // Display an error message if the file isn't opened properly
        std::cerr << "Error opening file: " << filename << std::endl;
        return;
    }

    std::string itemName;
    // Read items from the file and update the frequency map
    while (inputFile >> itemName) {
        itemFrequency[itemName]++;
    }

    // Close the input file
    inputFile.close();
}

void GroceryTracker::saveDataToFile() {
    // Open the "frequency.dat" file to write to
    std::ofstream outputFile("frequency.dat");
    for (const auto& entry : itemFrequency) {
        // Write the item and frequency to the file
        outputFile << entry.first << " " << entry.second << std::endl;
    }
    // Close the output file
    outputFile.close();
}

void GroceryTracker::searchItem() {
    std::string searchItem;
    // Prompt the user for the item they want to search for in options 1
    std::cout << "Enter the item to search for: ";
    std::cin >> searchItem;

    // Search for the item in the map
    auto it = itemFrequency.find(searchItem);
    if (it != itemFrequency.end()) {
        // If found, display the frequency
        std::cout << "Frequency of " << searchItem << ": " << it->second << std::endl;
    }
    else {
        // If not found, inform the user it's not in the records
        std::cout << searchItem << " not found in records." << std::endl;
    }
}

void GroceryTracker::printFrequencyList() {
    // Display the frequency list of all items
    for (const auto& entry : itemFrequency) {
        std::cout << entry.first << " " << entry.second << std::endl;
    }
}

void GroceryTracker::printHistogram() {
    // Display a histogram representation of item frequencies
    for (const auto& entry : itemFrequency) {
        std::cout << entry.first << " ";
        // Print asterisks for each occurrence of the item
        for (int i = 0; i < entry.second; ++i) {
            std::cout << "*";
        }
        std::cout << std::endl;
    }
}

void GroceryTracker::runMenu() {
    int choice;
    // Display the main menu and process user input
    do {
        std::cout << "\nMenu:\n"
            << "1. Search for an item\n"
            << "2. Display frequency list\n"
            << "3. Display histogram\n"
            << "4. Exit\n"
            << "Enter your choice: ";

        std::cin >> choice;

        switch (choice) {
        case 1:
            searchItem();
            break;
        case 2:
            printFrequencyList();
            break;
        case 3:
            printHistogram();
            break;
        case 4:
            std::cout << "Exiting program.\n";
            break;
        default:
            std::cout << "Invalid choice. Please try again.\n";
        }
    } while (choice != 4);

    // Save data to file before exiting the program
    saveDataToFile();
}

int main() {
    // Create an instance of the GroceryTracker class
    GroceryTracker groceryTracker;

    // Process the input file
    groceryTracker.processFile("CS210_Project_Three_Input_File.txt");

    // Run the main menu
    groceryTracker.runMenu();

    return 0;
}
