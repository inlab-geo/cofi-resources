# Python bindings for C++ code

There are different options to call libraries implemented in C++ from python

pybind11 - [https://github.com/pybind/pybind11](https://github.com/pybind/pybind11)
- Primarily working with C++ code and looking for a "natural" way to expose it to Python.
- The C++ code makes extensive use of modern C++ features.
- Performance is a significant concern.
- Comfortable with a C++ build process.
- Example: [https://reaktoro.org/](https://reaktoro.org/)
  
ctypes - [https://docs.python.org/3/library/ctypes.html](https://docs.python.org/3/library/ctypes.html):
- Interfacing with a simple C library.
- A pure Python solution without a separate compilation step for simple cases.
- Performance is not the top priority.
- No complex C++ dependencies or need to wrap C++ classes directly.
- Example: [https://github.com/inlab-geo/pyhk](https://github.com/inlab-geo/pyhk)

SWIG - [https://www.swig.org/](https://www.swig.org/):
- Need bindings for multiple target languages in addition to Python.
- Working with a large C/C++ codebase where automatic generation can save significant effort (though complexity might arise for advanced features).
- The project already uses SWIG
- Example: [https://github.com/anu-ilab/pyrjmcmc](https://github.com/anu-ilab/pyrjmcmc)


| Feature             | SWIG                                     | ctypes                                  | pybind11                                  |
| ------------------- | ---------------------------------------- | --------------------------------------- | ----------------------------------------- |
| **Language Support** | Many (Python, Java, etc.)                | Primarily C                             | Primarily C++                                    |
| **Binding Style** | Interface files (`.i`)                   | Python code (manual definitions)        | C++ code                                  |
| **C++ Support** | Good, but can require complex configuration | Limited, best for C-style interfaces   | Excellent (classes, templates, etc.)      |
| **Ease of Use** | Moderate (learning `.i` syntax)          | Easy for simple C, hard for complex C++ | Easy for C++ developers                 |
| **Performance** | Can have overhead                       | Generally slower                          | Good                                      |
| **Built-in** | No                                       | Yes                                     | No (header-only library)                  |
| **Compilation** | Requires SWIG compiler and C++ compiler  | Optional (runtime in inline mode)       | Requires C++ compiler                     |
| **Maturity** | Very mature                              | Mature                                  | Relatively newr, but stable and growing |
| **Use Cases** | Large projects, multi-language needs     | Simple C libraries, quick prototyping   | Modern C++ libraries, performance-critical bindings |
