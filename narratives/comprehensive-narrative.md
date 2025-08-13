# Comprehensive Capstone Narrative

## Introduction
The artifact selected for enhancement throughout the CS-499 capstone project is the **Grocery Tracker** application, originally developed in CS-250: Software Development Lifecycle. This console-based C++ program tracks the frequency of grocery items from an input file, supports user searches, and displays results. Across three milestones, the artifact was strategically enhanced in the areas of **software design and engineering**, **algorithms and data structures**, and **databases**. These iterative improvements transformed the original single-class, flat-file program into a modular, performance-optimized, database-driven application that reflects professional software development practices.

As for course outcomes, I have now achieved all five:  
- **Outcome 1:** Demonstrated collaborative readiness through well-documented, modular code that is easy for others to follow.  
- **Outcome 2:** Developed professional-quality documentation and reflections throughout the project.  
- **Outcome 3:** Showed algorithmic thinking with performance-focused data structure enhancements.  
- **Outcome 4:** Applied industry practices, including object-oriented design and database integration.  
- **Outcome 5:** Embraced a security mindset when handling data and designing interactions with SQLite.  

These trends have shaped my view of software development not just as a technical task, but as an activity with real impact on people and systems.

---

## Enhancement One: Software Design and Engineering
The initial enhancement addressed software design by **refactoring the monolithic codebase** into three separate classes (`FileHandler`, `GroceryTracker`, and `MenuManager`), to enforce separation of concerns. Hard-coded strings were replaced with named constants, and duplicate logic was consolidated into a helper function. Defensive programming was introduced to handle invalid input gracefully, while case-insensitive tracking improved usability and data integrity.

**Key Outcomes Demonstrated:**  
- **Outcome 1:** Produced modular, well-documented code that is easier for others to read, maintain, and extend, enabling collaborative development.  
- **Outcome 3:** Designed and evaluated a modular computing solution, considering trade-offs between simplicity, scalability, and maintainability.  
- **Outcome 4:** Applied professional-grade design techniques to create a maintainable, reusable codebase.  
- **Outcome 5:** Added safeguards against invalid input and subtle data issues.

---

## Enhancement Two: Algorithms and Data Structures
The second enhancement focused on algorithmic efficiency and feature expansion. A `std::map` was replaced with a `std::unordered_map` to achieve average O(1) insertion and lookup times, improving scalability for larger datasets. A sorted frequency display feature was implemented by transferring entries into a vector and using a custom comparator with `std::sort`. This provided users with the ability to view results in descending order of frequency, adding practical value.

**Key Outcomes Demonstrated:**  
- **Outcome 3:** Applied algorithmic principles to optimize performance and evaluated trade-offs in container selection.  
- **Outcome 4:** Leveraged STL containers, lambda functions, and sorting algorithms to create a responsive, user-focused feature.

---

## Enhancement Three: Databases
The third enhancement replaced the flat-file storage system with a **SQLite database backend**. This included creating relational storage with item names as primary keys, implementing upsert logic within transactions, migrating legacy data into the database, and exposing new user options for direct database manipulation. These changes provided durability, consistency, and scalability to the application.

**Key Outcomes Demonstrated:**  
- **Outcome 2:** Produced professional documentation detailing database design choices, migration strategy, and usage instructions.  
- **Outcome 3:** Evaluated storage trade-offs and selected SQLite for its structured query capabilities and data integrity features.  
- **Outcome 4:** Applied industry-relevant database tools and techniques to improve persistence and reliability.  
- **Outcome 5:** Incorporated transactions, primary keys, and safe data migration processes to anticipate and prevent data integrity issues.

---

## Reflection on the Enhancement Process
Each enhancement built on the previous one, resulting in a cohesive and professional-grade artifact. The **software design improvements** created a strong foundation for scalability. The **algorithmic optimizations** enhanced performance and expanded functionality without sacrificing maintainability. Finally, the **database integration** elevated the application into a durable, real-world solution capable of handling persistent data reliably.

Across all enhancements, I demonstrated:
- The ability to collaborate effectively through clear, modular code and comprehensive documentation (Outcome 1).  
- Strong problem-solving skills rooted in algorithmic thinking and performance trade-offs (Outcome 3).  
- Application of professional software engineering and database integration practices (Outcome 4).  
- Professional-quality written communication in describing technical decisions and processes (Outcome 2).  
- A proactive security mindset when handling user input and persistent data (Outcome 5).  

This iterative enhancement process has reinforced not only my technical abilities but also my understanding of how thoughtful design, performance considerations, and secure data management work together to produce solutions that are both functional and impactful.

---

