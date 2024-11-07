# User-Access-Management-system-
To develop a User Access Management System based on the provided specifications, we need to build the following components:

Java Servlets to handle the backend logic for each of the processes (sign-up, login, request access, approval, etc.).
JSP Pages to provide the frontend interface for the users (employees, managers, admins) to interact with the system.
PostgreSQL Database to store the necessary information such as user details, software applications, and access requests.
Here’s a high-level structure of the program:
/user-access-management
  /src
    /com
      /uam
        - SignUpServlet.java
        - LoginServlet.java
        - SoftwareServlet.java
        - RequestServlet.java
        - ApprovalServlet.java
        - User.java (Model)
        - Software.java (Model)
        - Request.java (Model)
  /webapp
    /WEB-INF
      - web.xml
    - signup.jsp
    - login.jsp
    - createSoftware.jsp
    - requestAccess.jsp
    - pendingRequests.jsp
  /db
    - create_tables.sql
  /pom.xml (Maven project file)
Step-by-Step Development Process
1. Database Setup
Create the PostgreSQL tables to store users, software, and requests. Here’s the SQL script (create_tables.sql):
-- Create Users Table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL CHECK(role IN ('Employee', 'Manager', 'Admin'))
);

-- Create Software Table
CREATE TABLE software (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    access_levels VARCHAR(255) NOT NULL
);

-- Create Requests Table
CREATE TABLE requests (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    software_id INT REFERENCES software(id),
    access_type VARCHAR(50) NOT NULL CHECK(access_type IN ('Read', 'Write', 'Admin')),
    reason TEXT,
    status VARCHAR(50) NOT NULL CHECK(status IN ('Pending', 'Approved', 'Rejected'))
);
2. Maven Configuration (pom.xml)
Here's a basic pom.xml for managing dependencies and building the project:
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.uam</groupId>
    <artifactId>user-access-management</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <!-- Servlet API -->
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>4.0.1</version>
            <scope>provided</scope>
        </dependency>
        
        <!-- JSTL for JSP -->
        <dependency>
            <groupId>javax.servlet.jsp.jstl</groupId>
            <artifactId>javax.servlet.jsp.jstl-api</artifactId>
            <version>1.2.1</version>
            <scope>provided</scope>
        </dependency>

        <!-- PostgreSQL JDBC Driver -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <version>42.2.5</version>
        </dependency>

        <!-- Logging -->
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
            <version>2.13.3</version>
        </dependency>

        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>2.13.3</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Maven plugin for compiling the servlets -->
            <plugin>
                <groupId>org.apache.tomcat.maven</groupId>
                <artifactId>tomcat7-maven-plugin</artifactId>
                <version>2.2</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>run</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
3. Java Servlets and Model Classes
SignUpServlet.java
Handles the sign-up logic.
@WebServlet("/SignUpServlet")
public class SignUpServlet extends HttpServlet {
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        String username = request.getParameter("username");
        String password = request.getParameter("password");
        
        // Encrypt password and store user in the database (skip for simplicity)
        
        // Insert user into database
        String role = "Employee"; // Default role
        
        try (Connection conn = DatabaseConnection.getConnection()) {
            String sql = "INSERT INTO users (username, password, role) VALUES (?, ?, ?)";
            try (PreparedStatement stmt = conn.prepareStatement(sql)) {
                stmt.setString(1, username);
                stmt.setString(2, password);
                stmt.setString(3, role);
                stmt.executeUpdate();
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        
        response.sendRedirect("login.jsp");
    }
}
LoginServlet.java
Handles user login.
@WebServlet("/LoginServlet")
public class LoginServlet extends HttpServlet {
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        String username = request.getParameter("username");
        String password = request.getParameter("password");
        
        try (Connection conn = DatabaseConnection.getConnection()) {
            String sql = "SELECT * FROM users WHERE username = ? AND password = ?";
            try (PreparedStatement stmt = conn.prepareStatement(sql)) {
                stmt.setString(1, username);
                stmt.setString(2, password);
                ResultSet rs = stmt.executeQuery();
                
                if (rs.next()) {
                    String role = rs.getString("role");
                    HttpSession session = request.getSession();
                    session.setAttribute("username", username);
                    session.setAttribute("role", role);
                    
                    if ("Employee".equals(role)) {
                        response.sendRedirect("requestAccess.jsp");
                    } else if ("Manager".equals(role)) {
                        response.sendRedirect("pendingRequests.jsp");
                    } else if ("Admin".equals(role)) {
                        response.sendRedirect("createSoftware.jsp");
                    }
                } else {
                    response.sendRedirect("login.jsp?error=Invalid credentials");
                }
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
SoftwareServlet.java (For Admin)
Handles the creation of software applications.
@WebServlet("/SoftwareServlet")
public class SoftwareServlet extends HttpServlet {
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        String name = request.getParameter("name");
        String description = request.getParameter("description");
        String accessLevels = String.join(",", request.getParameterValues("access_levels"));
        
        try (Connection conn = DatabaseConnection.getConnection()) {
            String sql = "INSERT INTO software (name, description, access_levels) VALUES (?, ?, ?)";
            try (PreparedStatement stmt = conn.prepareStatement(sql)) {
                stmt.setString(1, name);
                stmt.setString(2, description);
                stmt.setString(3, accessLevels);
                stmt.executeUpdate();
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        
        response.sendRedirect("createSoftware.jsp");
    }
}
RequestServlet.java
Handles the access request submission by employees.
@WebServlet("/RequestServlet")
public class RequestServlet extends HttpServlet {
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        int softwareId = Integer.parseInt(request.getParameter("softwareId"));
        String accessType = request.getParameter("accessType");
        String reason = request.getParameter("reason");
        
        try (Connection conn = DatabaseConnection.getConnection()) {
            String sql = "INSERT INTO requests (user_id, software_id, access_type, reason, status) VALUES (?, ?, ?, ?, 'Pending')";
            try (PreparedStatement stmt = conn.prepareStatement(sql)) {
                HttpSession session = request.getSession();
                int userId = (int) session.getAttribute("userId"); // Assume user ID is stored in session
                
                stmt.setInt(1, userId);
                stmt.setInt(2, softwareId);
                stmt.setString(3, accessType);
                stmt.setString(4, reason);
                stmt.executeUpdate();
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        
        response.sendRedirect("requestAccess.jsp");
    }
}
ApprovalServlet.java
Handles the approval or rejection of access requests by managers.
@WebServlet("/ApprovalServlet")
public class ApprovalServlet extends HttpServlet {
    protected void do
