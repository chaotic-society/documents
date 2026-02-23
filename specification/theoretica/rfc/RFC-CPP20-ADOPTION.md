# RFC 002: C++20 Dual Adoption

## Abstract
An modern C++20 adoption strategy for Theoretica is presented, with a hybrid approach which ensures maximal compatibility with all applications of the library. This dual adoption strategy enables embedded systems support while leveraging modern C++ features in advanced modules, particularly for improved type constraints and template metaprogramming.

## Motivation
As C++20 is rapidly becoming the new industry standard for high-performance scientific computing, Theoretica needs to take advantage of key features to be a modern, powerful library. Theoretica currently targets C++14 as its minimum standard, which limits the use of modern C++ features that could significantly improve code clarity, type safety, and developer experience. However, an immediate full migration to C++20 would break compatibility with:

1. **Embedded systems** with older toolchains
2. **Legacy build systems** in research institutions and enterprises
3. **Constrained environments** where compiler upgrades are difficult or impossible

### Use Cases

**Embedded Systems Requirements:**
- Predictable memory usage and performance
- Minimal runtime overhead
- Long-term stability (10+ year lifecycles)
- Compiler versions often lag by several years
- Key modules: `algebra`, `complex`, `core`, `calculus`, `signal`

**HPC Requirements:**
- Latest compiler optimizations
- Modern language features for cleaner code
- Advanced type safety (concepts, constraints)
- Rapid development iteration
- Key modules: `autodiff`, `optimization`, ML integration & GPU acceleration (planned)

### Key C++20 Features

1. **Concepts** replace SFINAE, making template constraints readable:
```cpp
// C++14
template<typename Type, enable_vector<T> = true>
auto compute(const Type& v);

// C++20
template<VectorType Type>
auto compute(const Type& v);
```

2. **Ranges** provide composable, lazy algorithms:
```cpp
// C++14
for (size_t i = 0; i < v.size(); ++i)
    if (v[i] > 0)
        result.push_back(std::sqrt(v[i]));

// C++20
auto result = v | std::views::filter([](auto x) { return x > 0; })
                | std::views::transform([](auto x) { return std::sqrt(x); });
```

3. **Improved constexpr** enables more compile-time computation:
```cpp
// C++20 allows std::vector and more algorithms at compile time
constexpr auto compute() {
    std::vector<real> coeffs;
    // ... complex compile-time computation
    return coeffs;
}
```

4. **Template improvements** (abbreviated function templates, `requires` clauses):
```cpp
// C++20
void process(auto&& x) requires Numeric<decltype(x)> {
    // ...
}
```

On the other hand, C++14 remains sufficient for core linear algebra, complex numbers, and basic calculus operations. These modules prioritize:
- Maximum portability
- Minimal complexity
- Embedded systems compatibility

## Design

Supporting two different standards requires a clear policy to differentiate between compiler requirements.
This is accomplished by marking headers or specific features as C++20-only.

### Modules

Modules that must maintain C++14 compatibility:

| Module | Rationale |
|--------|-----------|
| `core` | Fundamental definitions, used everywhere |
| `algebra` | Linear algebra is core functionality |
| `complex` | Essential for scientific computing |
| `calculus` | Numerical integration/differentiation |
| `interpolation` | Common in embedded systems |
| `signal` | FFT used in embedded systems |

Modules that will be enhanced with C++20 features:

| Module | Rationale |
|--------|-----------|
| `autodiff` | Concepts significantly simplify multivariate differential operators |
| `optimization` | Multivariate optimization benefits from concepts |
| `polynomial` | Constexpr improvements for compile-time polynomials |
| `pseudorandom` | Ranges for sampling operations |
| `statistics` | Ranges improve data processing pipelines |
| `io` | data_table and formatting |

### Implementation Strategy

Standard detection will be implemented in `core/constants.h`, which will define the `THEORETICA_USE_CPP20` macro if C++20 support is available and enabled.
Library end users may disable C++20 features by defining the `THEORETICA_DISABLE_CPP20` macro which will disable the previous check. The `THEORETICA_USE_CPP20` macro will be used to enable or disable header files or specific enhancements, with suitable error messages when features requiring the standard are not supported.

Two main standard separations may be used: **header-level** (entire header is C++20-only) or **feature-level** (specific functions or classes within a header are C++20-only). The choice depends on the extent of C++20 features used in each module. For example, small enhancements in `statistics`, such as ranges support, may be feature-level, while differential operators in `autodiff` may require header-level separation due to heavy use of concepts. At the moment, no module-level separation may be needed at all.

In addition, specific features for embedded systems using C++14 may be implemented to further strengthen support for these platforms.

## Drawbacks

### Increased Complexity

**Risk:**
- Two different C++ standards to manage
- More complex CMakeLists.txt and Makefile
- Potential for configuration errors
- CI/CD must test both configurations

**Mitigation:**
- Clear documentation of module categories
- Automated testing of both C++14-only and C++20 builds

### Developer Confusion

**Risk:** Contributors unsure which standard to use for new code.

**Mitigation:**
- Clear guidelines in CONTRIBUTING.md
- Documentation specifies required standard

### Maintenance Burden

**Challenge:** Maintaining two code paths for some features.

**Mitigation:**
- Prioritize C++14 base implementation
- C++20 enhancements are opt-in, not required
- Automated tests for both paths

## Alternatives

### Alternative 1: Full C++20 Migration

**Pros:**
- Simpler, single standard everywhere
- Access to all modern features
- Cleaner codebase

**Cons:**
- Breaks embedded systems compatibility
- Excludes users with older compilers
- Not aligned with Theoretica's portability goals

**Decision:** Rejected. Portability is a core value of Theoretica.

### Alternative 2: Stay on C++14

**Pros:**
- Maximum compatibility
- Simpler build system
- No migration work needed

**Cons:**
- Miss out on C++20 benefits (concepts, ranges)
- Advanced modules harder to maintain (SFINAE complexity)
- Less attractive to contributors familiar with modern C++
- Future-proofing concerns

**Decision:** Rejected. Advanced modules would suffer significantly.

### Alternative 3: Separate Libraries

**Pros:**
- Clear separation
- Each library has one standard
- Users choose what to install

**Cons:**
- Fragmented ecosystem
- Packaging complexity
- Harder to maintain
- Confusing for users

**Decision:** Rejected. Single library with dual standards is simpler.

### Alternative 4: C++17 as Base

**Pros:**
- More features than C++14
- Structured bindings, if constexpr, etc.

**Cons:**
- Still excludes embedded systems (many stuck on C++14)
- Doesn't provide concepts (main C++20 benefit)
- Not enough advantage over C++14

**Decision:** Rejected. Minimal benefit vs. C++14.

## Unresolved

### Module Categorization
It remains to be determined which modules will be C++20-only, if any.
The specific implementation of C++20 features for each module needs a more refined program.

### Python Bindings
What standard of C++ should the Python bindings use? 

**Options:**
1. C++14 (for consistency with core)
2. C++20 (pybind11 works better with modern C++)
3. Match the bound module's standard

### Future Standards
The possibility of adopting even newer standards remains open. For this reason, a clearer policy should be defined.
Widespread adoption of newer standards may be a good adoption criterion.

## Implementation Plan
- [ ] Define macros for C++ standard detection in `constants.h`
- [ ] Update CMakeLists.txt and Makefile to support dual standard builds
- [ ] Port differential operators in `autodiff` to C++20 with concepts
- [ ] Port multivariate optimization in `optimization` to C++20 with concepts
- [ ] Implement C++20 ranges in `statistics` for data processing

## Testing Strategy
The dual standard could be tested using an automated CI/CD pipeline which uses an extended test matrix to build and test both C++14 and C++20 configurations. This ensures that both builds are working as intended.

## Documentation
Standard separation will be documented per header, with indications about header-level or feature-level C++20 requirements. The `CONTRIBUTING.md` file will include guidelines for contributors on which standard to use for new code and how to implement C++20 features when appropriate.

## Backwards Compatibility
The C++14 code path will be maintained as the baseline, ensuring that all existing functionality remains available to users with older compilers. C++20 features will be added as enhancements without breaking existing code, with the exception of `autodiff` and `optimization` which will be C++20-only due to heavy use of concepts.

## References
- [C++20 Standard](https://en.cppreference.com/w/cpp/20)
- [C++20 Concepts](https://en.cppreference.com/w/cpp/language/constraints)
- [C++20 Ranges](https://en.cppreference.com/w/cpp/ranges)
- [C++14 Standard](https://en.cppreference.com/w/cpp/14)
