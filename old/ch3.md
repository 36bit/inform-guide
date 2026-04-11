# Chapter 3: Your First Inform 6 Project

## Chapter 3.1: Anatomy of an Inform 6 Project: Dissecting the Building Blocks

Before we dive into crafting your first interactive fiction 
masterpiece, let's take a closer look at the essential components that 
make up an Inform 6 project. Understanding the structure and purpose 
of these elements will provide a solid foundation for your development 
journey.

### Source Code Files: The Blueprint of Your World

Source code files are the heart of your Inform 6 project. They contain 
the instructions written in the Inform 6 language that define the game 
world, characters, puzzles, and narrative. These files serve as the 
blueprint for your interactive story, dictating how the game behaves 
and responds to the player's actions.

#### Structure and Syntax:

Inform 6 source code is organized into routines, which are blocks of 
code that perform specific tasks. Each routine is enclosed within 
square brackets ([ ]) and has a name that identifies its purpose. 
Within routines, statements are used to execute actions, make 
decisions, and manipulate data. Inform 6 utilizes a unique syntax that 
combines natural language elements with programming constructs, making 
it relatively easy to read and understand.

#### Tips for Organization and Efficiency:

1. **Modularization**: Break down your code into smaller, well-defined 
routines. This improves code readability, maintainability, and reusability.

2. **Descriptive Naming**: Use meaningful names for routines and 
variables to make your code self-documenting and easier to understand.

3. **Comments**: Add comments to explain complex sections of code or 
logic. This will be helpful for both yourself and others who may work 
on your project in the future.

4. **Indentation**: Use consistent indentation to visually organize 
your code and improve readability.

#### Common Coding Practices and Patterns:

1. **Object-Oriented Design**: Inform 6 supports object-oriented 
programming principles, allowing you to define classes and objects to 
represent elements of your game world. This approach promotes 
modularity and code reuse.

2. **Message Passing**: Objects in Inform 6 communicate with each 
other through message passing, which allows for a flexible and dynamic 
game design.

3. **Action Handling**: Inform 6 provides a robust system for defining 
and handling actions that the player can take. You can define custom 
actions and specify how objects react to them.

By following these tips and utilizing common coding practices, you can 
write efficient, well-organized, and maintainable Inform 6 code.

### Library Files: A Treasure Trove of Pre-Built Functionality

Library files contain pre-written Inform 6 code that provides commonly 
used functionalities and game mechanics. These files save you from 
having to reinvent the wheel, allowing you to focus on the unique 
aspects of your game design.

#### Standard Library Files:

The Inform 6 standard library is maintained by David Griffith at 
https://gitlab.com/DavidGriffith/inform6lib. It comes with a set of 
standard library files that provide essential functionalities for 
creating interactive fiction games. These files include:

- **Parser.h**: Provides the parser that interprets the player's 
commands.

- **VerbLib.h**: Defines the standard actions that the player can 
take, such as "take," "drop," "go," and "examine."

- **Grammar.h**: Defines the grammar rules that the parser uses to 
understand the player's commands.

#### Including and Using Library Files:

You can include library files in your project using the `Include` 
directive. This effectively inserts the contents of the library file 
into your source code at the point of inclusion.

#### Modifying or Creating Custom Library Files:

While the standard library provides a solid foundation, you may need 
to modify existing library routines or create your own custom 
libraries to implement specific game mechanics or features. Inform 6 
allows you to replace existing library routines with your own 
definitions and create new library files that can be included in your 
project.

By leveraging the power of library files, you can streamline your 
development process and focus on the creative aspects of your game 
design.

### The Compiled Game File: From Blueprint to Playable Reality

The process of transforming your Inform 6 source code into a playable 
game file is called compilation. The Inform 6 compiler takes your 
source code and translates it into a format that can be understood and 
executed by an interpreter program.

#### Compiled File Formats:

Inform 6 can compile games into different formats, depending on the 
target Z-machine version. The most commonly used formats are:

- **.z5**: The standard format for Inform 6 games, compatible with 
most interpreters.

- **.z8**: An extended format that allows for larger and more complex 
games.

- **.gblorb**: A format that can include multimedia resources such as 
images and sounds.

#### Tools and Commands for Compilation:

The inform command-line tool is used to compile Inform 6 source code. 
You can specify various options and switches to control the 
compilation process, such as the target Z-machine version and 
debugging settings.

#### Troubleshooting Compilation Errors:

The Inform 6 compiler will generate error messages if it encounters 
problems in your source code. These messages can help you identify and 
fix the errors.

### Additional Components: Enhancing Your World

Beyond the core elements of source code, library files, and the 
compiled game file, Inform 6 projects can incorporate various 
additional components:

- **Media Files**: You can include images, sounds, and other 
multimedia resources to enhance the player experience. These files are 
typically integrated using extensions or libraries that provide the 
necessary functionality.

- **Extensions**: Inform 6 has a vibrant community that contributes 
extensions and libraries to expand the capabilities of the language. 
These extensions can provide features such as advanced text 
formatting, menu systems, and support for multimedia.

By incorporating these additional components, you can create richer 
and more immersive interactive fiction experiences.

### Project Management and Organization: Keeping Your World in Order

As your Inform 6 project grows in complexity, effective project 
management becomes crucial. Here are some best practices:

- **Version Control**: Use a version control system like Git to track 
changes to your source code, collaborate with others, and revert to 
previous versions if necessary.

- **Modularization**: Organize your code into well-defined modules and 
libraries to improve maintainability and reusability.

- **Documentation**: Document your code and design decisions to help 
yourself and others understand your project.

- **Collaboration Tools**: If you are working on a project with a 
team, utilize collaboration tools like shared repositories and issue 
trackers to facilitate communication and coordination.

By adopting these practices, you can ensure that your Inform 6 project 
remains organized and manageable, even as it grows in size and 
complexity.

This comprehensive examination of the anatomy of an Inform 6 project 
provides a solid foundation for both new and experienced developers. 
By understanding the different components and their roles, you can 
approach your IF projects with greater confidence and efficiency. As 
you delve deeper into the world of Inform 6 programming, remember that 
this guide and the vibrant IF community are always here to support 
your creative journey.

## Chapter 3.2: Creating a New Inform 6 Project: Laying the Groundwork for Your Interactive Story

With a firm grasp of the essential components of an Inform 6 project, 
it's time to embark on the exciting journey of creating your own 
interactive fiction game. This section will guide you step-by-step 
through setting up the necessary files and directory structure for a 
new project, laying the groundwork for your narrative masterpiece.

### Building the Foundation: Step-by-Step Guide

1. Create a Project Directory: Choose a suitable location on your 
computer and create a new directory to house your Inform 6 project. 
Give it a descriptive name that reflects the theme or title of your 
game.

2. Main Source File: Inside the project directory, create a new text 
file named main.inf. This file will serve as the primary source code 
file for your game, containing the main routine and other essential 
definitions.

3. Additional Source Files (Optional): Depending on the complexity of 
your project, you may choose to create additional source code files to 
organize your code. For example, you could create separate files for 
game objects, rooms, and functions.

4. Library Files: If your game utilizes libraries or extensions beyond 
the standard Inform 6 library, create a subdirectory named lib within 
your project directory. Place the necessary library files in this 
directory.

5. Include Directives: In your main.inf file, use the Include 
directive to incorporate the required library files. For example, to 
include the standard Inform 6 library files, add the following lines 
at the beginning of main.inf:

    Include "Parser";
    Include "VerbLib";
    Include "Grammar";

If you are using additional libraries or extensions, include them 
using the appropriate path relative to the lib directory.

### Understanding the Ecosystem: The Role of Each Component

**Project Directory**: This directory serves as the central hub for 
all your project files, keeping them organized and easily accessible.

**main.inf**: This file acts as the entry point for your game. It 
contains the `Main` routine, which is the first routine executed when 
the game starts. Additionally, `main.inf` typically includes other 
essential definitions and directives, such as global variables and 
object declarations.

**Additional Source Files**: These files allow you to further 
modularize your code, separating different aspects of your game logic 
and design into distinct files for better organization and 
maintainability.

**lib Directory**: This directory stores any additional libraries or 
extensions used by your project. Keeping these files separate from 
your main source code promotes organization and clarity.

**Include Directives**: These directives tell the Inform 6 compiler to 
incorporate the contents of the specified library files into your 
source code, providing access to pre-built functionalities and game 
mechanics.

By understanding the role of each file and directory, you can 
effectively manage your Inform 6 project and ensure a smooth 
development process.

*** Example: Setting Up a Simple Project

Let's illustrate the process by setting up a new project named "MyFirstGame":

1. Create a new directory named `MyFirstGame`.

2. Inside the `MyFirstGame` directory, create a new text file named 
`main.inf`.

Open `main.inf` in a text editor and add the following code:

    Constant Story "My First Game";
    Constant Headline "An Interactive Adventure";

    Include "Parser";
    Include "VerbLib";
    Include "Grammar";

    [ Initialise;
        location = LivingRoom;
        "You find yourself in a cozy living room.";
    ];

    Object LivingRoom "Living Room"
    with description "The living room is warm and inviting.";

3. Save the `main.inf` file.

This simple project structure establishes the foundation for your 
game. The `main.inf` file includes the standard library files and 
defines the initial state of the game, placing the player in the 
"Living Room" object.

### Beyond the Basics: Expanding Your Project

As your game grows in complexity, you can expand your project 
structure to accommodate additional components:

- Media Files: Create a subdirectory named media to store images, 
sounds, and other multimedia resources. You can then integrate these 
resources into your game using appropriate extensions or libraries.

- Custom Libraries: If you create your own libraries or extensions, 
store them in the lib directory for easy access and organization.

Remember to maintain a clear and consistent structure as your project 
grows, ensuring that all files and directories are well-organized and 
serve a specific purpose within your game's ecosystem.

### A World of Possibilities: Embracing the Creative Potential

Setting up a new Inform 6 project is just the first step on an 
exciting journey of interactive fiction creation. With the foundation 
laid, you are now ready to unleash your creativity and build a 
captivating world filled with intriguing characters, challenging 
puzzles, and a compelling narrative. As you progress through this 
guide, you will acquire the knowledge and skills to transform your 
initial project structure into a fully realized interactive story, 
ready to be shared and enjoyed by players around the world.

## Chapter 3.3: Writing the "Hello World" Program in Inform 6: Your First Steps into Interactive Storytelling

The "Hello World" program is a time-honored tradition in the world of 
programming, serving as a gentle introduction to a new language and 
its basic syntax. In this section, we will craft your very own "Hello 
World" program in Inform 6, taking your first steps into the realm of 
interactive fiction creation.

Building the Program: A Step-by-Step Guide

1. **Open your `main.inf` file**: This file, created in the previous 
section, will house your "Hello World" program.

2. **Define the Main Routine**: Every Inform 6 program requires a 
`Main` routine, which serves as the starting point for execution. Add 
the following code to your `main.inf` file:

    [ Main;

    ];

3. **Print "Hello, world!"**: Within the Main routine, we will use the 
print statement to display "Hello, world!" on the screen. Add the 
following line within the square brackets:

    print "Hello, world!";

4. Save your `main.inf` file: Your "Hello World" program is now 
complete!

### Understanding the Fundamentals: Basic Inform 6 Syntax and Concepts

Now, let's delve deeper into the code and explore the fundamental 
concepts of Inform 6 that are showcased in this simple program:

- **Routines*: Routines are the building blocks of Inform 6 programs. 
They are named blocks of code that perform specific tasks. The `Main` 
routine is a special routine that serves as the program's entry point.

- **Statements**: Statements are instructions within routines that 
tell the program what to do. The `print` statement is used to display 
text on the screen.

- **Printing**: Inform 6 uses the `print` statement to output text to 
the screen. Text to be printed is enclosed within double quotation 
marks (`" "`).

In the "Hello World" program, the `Main` routine contains a single 
`print` statement that displays "Hello, world!" on the screen. This 
demonstrates the basic structure of Inform 6 programs and introduces 
the fundamental concepts of routines, statements, and printing.

### Best Practices and Troubleshooting: A Smooth Start to Your Journey

As you begin your Inform 6 coding journey, keep these best practices 
in mind:

- **Semicolons**: Statements in Inform 6 are separated by semicolons 
  (;).

    Indentation: Use consistent indentation to improve code readability.

    Comments: Add comments to explain your code and make it easier to understand.

If you encounter any issues while writing or compiling your "Hello 
World" program, carefully review the error messages generated by 
Inform 6. These messages often provide valuable clues about the nature 
of the problem and how to fix it. Additionally, consult the Inform 6 
documentation or seek assistance from the online community forums and 
discussion groups. 

### Beyond "Hello World": A Foundation for Interactive Fiction

While the "Hello World" program is simple, it introduces the 
fundamental building blocks of Inform 6 programming. By understanding 
routines, statements, and printing, you have laid the groundwork for 
creating more complex and engaging interactive fiction experiences. In 
the following chapters, we will build upon these basic concepts and 
explore the full power of Inform 6 as a tool for crafting immersive 
and interactive narratives.