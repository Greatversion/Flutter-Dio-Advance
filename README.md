 ---
# Flutter HTTP Network Service (Dio + Connectivity) Documentation

## Introduction

This documentation outlines the steps for building a scalable and production-ready HTTP network service in Flutter using the `Dio` HTTP client. This solution includes error handling, network connectivity checks, and support for various HTTP methods (GET, POST, PUT, DELETE). The approach is designed for real-world production use, ensuring resilience and reliability for all network requests.

---

## Prerequisites

To follow this guide, ensure that you have:
- Flutter installed on your machine.
- A basic understanding of Dart and Flutter.
- An internet connection to install necessary packages.

---

## Dependencies

Add the following dependencies to your `pubspec.yaml` file:

```yaml
dependencies:
  dio: ^5.1.2               # HTTP client for making network requests.
  connectivity_plus: ^4.0.2 # Used to check internet connectivity.
```

Run the following command to install the dependencies:

```bash
flutter pub get
```

---

## Network Service Implementation

### 1. **Create `network_service.dart`**

The `NetworkService` class is the main component that handles all HTTP requests and responses, manages error handling, and checks for connectivity.

#### Code:

```dart
import 'package:dio/dio.dart';
import 'package:connectivity_plus/connectivity_plus.dart';

class NetworkService {
  late Dio dio;

  // Base URL of the API (you can set this to your API endpoint)
  final String baseUrl = "https://your-api-endpoint.com";

  NetworkService() {
    dio = Dio(
      BaseOptions(
        baseUrl: baseUrl,
        connectTimeout: Duration(seconds: 10), // Timeout for connection
        receiveTimeout: Duration(seconds: 15), // Timeout for receiving data
      ),
    );

    // Adding interceptors to handle global behaviors like logging and error handling
    dio.interceptors.add(
      InterceptorsWrapper(
        onRequest: (options, handler) {
          // You can add additional headers here if needed
          options.headers["Accept"] = "application/json";
          return handler.next(options);
        },
        onResponse: (response, handler) {
          // Logging or other global response handling
          print("Response: ${response.data}");
          return handler.next(response);
        },
        onError: (DioError e, handler) async {
          // Handle errors globally
          return handler.next(await _handleError(e));
        },
      ),
    );
  }

  // GET request
  Future<Response?> get(String endpoint, {Map<String, dynamic>? queryParams}) async {
    return await _performRequest(() => dio.get(endpoint, queryParameters: queryParams));
  }

  // POST request
  Future<Response?> post(String endpoint, {Map<String, dynamic>? data}) async {
    return await _performRequest(() => dio.post(endpoint, data: data));
  }

  // PUT request
  Future<Response?> put(String endpoint, {Map<String, dynamic>? data}) async {
    return await _performRequest(() => dio.put(endpoint, data: data));
  }

  // DELETE request
  Future<Response?> delete(String endpoint, {Map<String, dynamic>? data}) async {
    return await _performRequest(() => dio.delete(endpoint, data: data));
  }

  // Centralized request handler to manage retries, connectivity check, and error handling
  Future<Response?> _performRequest(Future<Response> Function() request) async {
    if (await _isConnected()) {
      try {
        return await request();
      } catch (e) {
        print("Error: $e");
        rethrow;
      }
    } else {
      throw DioError(
        requestOptions: RequestOptions(path: ''),
        error: 'No internet connection',
        type: DioErrorType.connectionError,
      );
    }
  }

  // Error handling logic
  Future<DioError> _handleError(DioError error) async {
    if (error.type == DioErrorType.connectionTimeout || error.type == DioErrorType.receiveTimeout) {
      return DioError(
        requestOptions: error.requestOptions,
        error: 'Connection Timeout! Please try again later.',
        type: DioErrorType.connectionTimeout,
      );
    } else if (error.type == DioErrorType.badResponse) {
      if (error.response?.statusCode == 400) {
        return DioError(
          requestOptions: error.requestOptions,
          error: 'Bad Request! Check the parameters.',
          type: DioErrorType.badResponse,
        );
      } else if (error.response?.statusCode == 401) {
        return DioError(
          requestOptions: error.requestOptions,
          error: 'Unauthorized! Please log in again.',
          type: DioErrorType.badResponse,
        );
      } else if (error.response?.statusCode == 500) {
        return DioError(
          requestOptions: error.requestOptions,
          error: 'Internal Server Error! Please try again later.',
          type: DioErrorType.badResponse,
        );
      } else {
        return DioError(
          requestOptions: error.requestOptions,
          error: 'Something went wrong! Please try again.',
          type: DioErrorType.badResponse,
        );
      }
    } else if (error.type == DioErrorType.cancel) {
      return DioError(
        requestOptions: error.requestOptions,
        error: 'Request was cancelled!',
        type: DioErrorType.cancel,
      );
    } else if (error.type == DioErrorType.connectionError) {
      return DioError(
        requestOptions: error.requestOptions,
        error: 'No Internet connection! Check your network.',
        type: DioErrorType.connectionError,
      );
    } else {
      return DioError(
        requestOptions: error.requestOptions,
        error: 'Unexpected error occurred! Please try again.',
        type: DioErrorType.unknown,
      );
    }
  }

  // Check if the device has internet connectivity
  Future<bool> _isConnected() async {
    var connectivityResult = await Connectivity().checkConnectivity();
    return connectivityResult != ConnectivityResult.none;
  }
}
```

### 2. **Key Concepts and Features:**

- **Dio Initialization**: A `Dio` instance is initialized with a base URL, and timeouts for both connection and receiving data.
- **Interceptors**: Used for logging and global request/response error handling. This is useful for adding headers, handling retries, or logging API calls.
- **GET, POST, PUT, DELETE Methods**: Methods are available for the most commonly used HTTP requests, including query parameters and request body support.
- **Centralized Error Handling**: The `_handleError` method classifies different network errors such as timeouts, connectivity issues, and bad HTTP responses (like 400, 401, 500).
- **Network Connectivity Check**: The `_isConnected` method uses the `connectivity_plus` plugin to check for internet availability before making any request.
- **Timeouts**: Both connection and receive timeouts are defined to handle cases where the server is slow or unresponsive.

---

## Error Handling

The `NetworkService` class has a comprehensive error handling strategy:

- **Timeout Handling**: The network request will throw a connection or receive timeout error if the request takes too long.
- **Bad Responses**: The class handles typical 4xx and 5xx HTTP responses and provides specific error messages for cases like:
  - 400 Bad Request
  - 401 Unauthorized
  - 500 Internal Server Error
- **No Internet Connection**: If there is no internet connection, the `_isConnected()` function will prevent any network request from being made and throw an error.
- **Request Cancellation**: If a request is canceled for any reason, the class will handle it appropriately.

---

## Usage Example

### Example of making a GET request in your Flutter app:

```dart
import 'package:flutter/material.dart';
import 'network_service.dart';

void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: HomeScreen(),
    );
  }
}

class HomeScreen extends StatefulWidget {
  @override
  _HomeScreenState createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  final NetworkService _networkService = NetworkService();

  @override
  void initState() {
    super.initState();
    fetchData();
  }

  Future<void> fetchData() async {
    try {
      final response = await _networkService.get('/endpoint'); // Replace '/endpoint' with your API endpoint
      if (response != null) {
        print('Response data: ${response.data}');
      }
    } catch (e) {
      print('Error occurred: $e');
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Network Service Demo'),
      ),
      body: Center(
        child: Text('Check your console for network logs.'),
      ),
    );
  }
}
```

### Key Points:
- Replace `'/endpoint'` with your API endpoint.
- When the `fetchData` function is called, it will make a GET request using the `NetworkService` class, and any errors or responses will be logged to the console.

---

## Conclusion

This documentation demonstrates how to build a robust and scalable network service in Flutter using Dio and connectivity checks. This service supports handling different error cases, including timeouts, network unavailability, and various HTTP response errors, making it well-suited for production environments.

You can easily extend the `NetworkService` class by

 adding more features such as token management, caching, and more depending on your application’s needs.
