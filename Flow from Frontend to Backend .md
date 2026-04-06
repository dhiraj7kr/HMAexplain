This is a classic N-Tier architecture (Frontend -> Controller -> Business Layer -> Database). Here is the comprehensive, step-by-step execution flow of how a single user action in the browser results in data being fetched from your SQL Server.

---

### **Step 1: The Trigger (Frontend)**
**File:** `DetailedBookingReport.tsx` (React)
* **User Action:** The user clicks "Search" or changes a filter (Date/Status).
* **Function:** `this.DetailedBookingReport(...)`
* **Action:** The React component gathers state values (`FromDate`, `ToDate`, `BookingId`, etc.), encrypts them using a constant key, and sends a **GET** request to the URL defined in your constants: `https://localhost:44333/api/Reports/GetDetailedBookingReportPagination`.

---

### **Step 2: The Gateway (API Controller)**
**File:** `ReportsController.cs`
**Function:** `GetDetailedBookingReportPagination(string Param)`
* **Job:** This is the entry point to the backend.
    1.  **Decrypts** the incoming URL parameters.
    2.  **Parses** the strings into correct C# types (e.g., converting "true" to a `bool`, and date strings to `DateTime?`).
    3.  **Calls the Business Layer:** It invokes the method in the BAL (Business Allocation Layer) interface.
* **Code Line:** `return EncryptedData(_reports.GetDetailedBookingReportPagination(...));`

---

### **Step 3: The Business Logic (BAL)**
**File:** `ReportsBAL.cs` (Implementation of `IReportsBAL`)
**Function:** `GetDetailedBookingReportPagination(...)`
* **Job:** This layer handles the "How" of data retrieval.
    1.  **Connection:** It fetches the connection string named `"HMASql"` from your configuration.
    2.  **Command Setup:** It creates a `SqlCommand` and explicitly sets its type to `CommandType.StoredProcedure`.
    3.  **Mapping Parameters:** It maps the C# variables to the specific SQL parameters expected by the database (e.g., mapping `company` to `@company`).
    4.  **Database Execution:** It uses a `SqlDataAdapter` to execute the procedure and fill a `DataSet` (`ds`).
    5.  **Serialization:** It converts the `DataSet` (a complex C# object) into a **JSON string** using `JsonConvert.SerializeObject(ds)`.

---

### **Step 4: The Data Engine (SQL Server)**
**Object:** Stored Procedure `[dbo].[USP_Rpt_Get_DetailedBookingReportPagination]`
* **Job:** This is where the heavy lifting happens. The procedure has three logical branches based on the input:
    * **Branch A (Export Excel):** Returns all matching records at once so the user can download a full file.
    * **Branch B (Pagination):** Uses `OFFSET` and `FETCH NEXT` to return only 10, 50, or 100 rows at a time. This keeps the UI fast.
    * **Branch C (Booking ID Search):** If a specific ID is provided, it ignores dates and searches specifically for that `HMARequestId`.
* **Data Sources:** It joins three main tables:
    1.  `tblTravelRequest`: Main booking info.
    2.  `tblPassengers`: Passenger-specific details.
    3.  `UserMasterDataTable`: Employee details (Designation, Band, Dept).
    4.  `tblInvoiceDetails`: Payment/Invoice info.

---

### **Step 5: The Return Journey**
1.  **SQL** returns the result set to the **BAL**.
2.  **BAL** serializes it to JSON and returns it to the **Controller**.
3.  **Controller** encrypts that JSON and sends it back over HTTP to **React**.
4.  **React** (`DetailedBookingReport.tsx`) receives the response in the `.done()` callback of the AJAX call.
5.  **React** decrypts the data and updates `this.state.Data`.
6.  **UI Updates:** The `BootstrapTable` detects the state change and automatically re-renders the rows on your screen.



### **Summary Table**

| Layer | File/Object | Key Responsibility |
| :--- | :--- | :--- |
| **Frontend** | `DetailedBookingReport.tsx` | UI State & AJAX Request |
| **Constants** | `Constants.json` | Stores the API URL & Encryption Keys |
| **Controller** | `ReportsController.cs` | Request Decryption & Routing |
| **Business (BAL)** | `ReportsBAL.cs` | SQL Parameter Mapping & Execution |
| **Database** | `Stored Procedure` | Data Joining, Filtering, and Pagination |

