Analyze functionality and dependencies of specified controller/service and implement comprehensive integration tests for it to be used by developers to validatate and maintain the correctness of controller/service during adding new features or refactoring existing code.

---

## Variable Placeholders

| Placeholder | Description | Example value |
|---|---|---|
| `{{language}}` | [Language for implementation] | [Java, TypeScript, Python] |
| `{{framework_or_library}}` | [Library or framework should be used to implement the tests] | [Spring Boot Test, Mockito, vitest] |
| `{{specific_tools}}` | [List of any explicit tools or approaches that should be used] | [MockMvcTester, Testcontainers, AssertJ or React Testing Library, MSW] |

---

## Output Format Instruction

Follow common naming convention for itegration tests for given project (for example, for Java/Kotlin projects check pom.xml), for example SomeTestIT.java.
**Structure tests** by endpoint or purpose using nested and/or parametrized tests.
**Limit using comments** to only very specific and complicated cases.
Limit number of lines to approx 1000 and split into several files if this limit is exceeded. For example: OrderControllerPutOrderIT, OrderControllerDeleteOrderIT

---

## Prompt Body

You are senior {{language}} software engineer with deep expertise in writing **integration tests** (IT) using {{framework_or_library}}.
Analyze {{controller_or_service}} and write the integration tests for each endpoint or public/open method. Use the following steps end ensure that **tests are robust and fully cover {{controller_or_service}} functionality**.
- Discover and follow project's naming conventions, fallback to common best practices when not set in project yet.
- Discover existing ITs and if they exist consider following the same pattern, but **only if it's correct**. It means, avoid re-using when IT is written using unit test approach or test mocks everything.
- Identify and prepare useful **test data** which do not conflict with other tests. Reuse it when possible.
- Implement tests according to [Output Format Instructios](#output-format-instruction)
- Tests must contain both positive and negative scenarios.
- Mock only dependency that can't be easily wired/set without mocking, for example downstream calls.
- Assert only specific expected values - never assert that a value is merely non-null or non-empty when a concrete value is available from seed data.
- Use {{specific_tools}} for writing and verifying tests
- Ask clarifying questions before writing tests when in doubt.
