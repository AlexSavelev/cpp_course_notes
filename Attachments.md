# Verbose class
```cpp
#pragma once

#include <iostream>

struct Verbose {
  Verbose() {
    std::cout << "Verbose " << id << " default constructed\n";
  }

  Verbose(const Verbose&) {
    std::cout << "Verbose " << id << " copy constructed\n";
  }

  Verbose(Verbose&&) noexcept {
    std::cout << "Verbose " << id << " move constructed\n";
  }

  Verbose& operator=(const Verbose& other) {
    std::cout << "Verbose " << id << " copy assigned with " << other.id << "\n";
    return *this;
  }
  
  Verbose& operator=(Verbose&& other) {
    std::cout << "Verbose " << id << " move assigned with " << other.id << "\n";
    return *this;
  }

  ~Verbose() {
    std::cout << "Verbose " << id << " destructed\n";
  }

 private:
  static std::size_t GetNext() {
    static std::size_t i = 0;
    ++i;
    return i;
  }

  std::size_t id{GetNext()};
};
```
