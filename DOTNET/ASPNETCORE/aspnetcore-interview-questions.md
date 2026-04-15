# ASP.NET Core Interview Questions

Source: `net-interview-questions.pdf` by Anton Martyniuk (Microsoft MVP).

1. Explain how routing works in ASP.NET Core
2. What is middleware and in what order do they execute?
3. How can you stop other middlewares from executing?
4. What is the difference between MVC and Razor Pages?
5. Name 3 ways to create middleware
6. Explain how `appsettings.json` configuration layering works
7. What is the difference between Singleton, Scoped, Transient services?
8. How to use Scoped service inside a Singleton service?
9. How to execute code when application is starting and stopping?
10. What is a Background Service?
11. Name a few ways to read data from `appsettings.json` configuration
12. What is the Options Pattern?
13. Name the use cases for using `ISnapshotMonitor` and `IOptionsMonitor`
14. How to validate configuration?
15. What is the difference between DataAnnotations and FluentValidation?
16. Explain action filters and how you might use them with controllers
17. Why are Minimal APIs faster than Controllers?
18. How to add authorization to the project?
19. How to add authorization to all Controller's methods except one?
20. How would you implement Log-In functionality?
21. Explain how JWT tokens work
22. Explain Refresh tokens and how they work
23. How would you implement an application that allows access to certain resources if a user has specific permissions?
24. What is `HostedService` used for?
25. Explain the difference between `PeriodicTimer` and `await Task.Delay()`
26. What is HSTS?
27. How to return a file from an API Endpoint?
28. How to accept a file in an API Endpoint?
29. How to access query string parameters in an API Endpoint?
30. How to get current logged-in user information?
31. How to inject dependencies in Minimal APIs?
32. How would you structure your Minimal API endpoints?
33. What is Output Caching?
34. Difference between `IMemoryCache` and `IDistributedCache`?
35. Explain how HybridCache/FusionCache works
36. What Caching patterns do you know?
37. What is Rate Limiting used for and what types do you know?
38. How to invalidate data in OutputCache?
39. How to implement API versioning?
40. How to add API versioning to the existing API if you're not allowed to change the API URLs?
41. What is Swagger used for?
42. How to add documentation of endpoints, models and fields in Swagger?
43. How to get a connection string from the configuration?
44. How can you deploy an ASP.NET Core application?
45. How to configure logging in ASP.NET Core?
46. What are the `IHostApplicationLifetime` events and how can you attach callbacks to each?
47. How can you stop a `BackgroundService` from execution?
48. What are the default return types you can use in controller actions?
49. What is model binding and how does it work in ASP.NET Core controllers?
50. Explain the difference between `[FromBody]`, `[FromQuery]`, `[FromRoute]`, and `[FromForm]` attributes
51. What is a custom model-binder class and how do you implement one?
52. How do you enable or disable a controller or action? Which attribute is used?
53. What is Method Injection in a controller and how does it differ from constructor injection?
54. How do you implement validation in controller actions? How do you handle invalid models?
55. Explain the lifecycle of a controller in ASP.NET Core. When is it instantiated and disposed?
56. What's the difference between `ControllerBase` and `Controller`? When would you use each?
57. How can you handle exceptions globally for all controllers?
58. What is structured logging and why should you use it?
59. How can you log request and response data for incoming HTTP requests?
60. How would you handle sensitive data in logs to ensure security and compliance (e.g., GDPR)?
