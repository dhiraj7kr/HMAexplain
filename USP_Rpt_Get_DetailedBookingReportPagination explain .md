### 1. The "Logic Switch" (C# BAL)
Inside `ReportsBAL.cs`, there is a specific piece of logic that controls how the database is queried.

* **Priority Search:** If a user enters a `BookingId`, the code intentionally sets `FromDate`, `ToDate`, and `Status` to `null` or empty.
* **Why?** This prevents a common bug where a user searches for a specific ID but fails to find it because the "Date Range" filter (which defaults to the last 30 days) excludes that old booking.



---

### 2. The SQL Architecture (Stored Procedure)
The stored procedure `USP_Rpt_Get_DetailedBookingReportPagination` uses a "Temporary Table" strategy (`#templatesttable`). Here is why:

#### **A. Phase 1: Data Gathering (The Temp Table)**
It first pulls raw data from `tblTravelRequest` and `tblPassengers`.
* It filters by `@company`, `@fromDate`, and `@toDate`.
* It calculates `IsCovidDeclared` (defaulting to 'No' if null).
* It handles **Guest Bookings** (Travelling Type 4) vs. **Employee Bookings**.

#### **B. Phase 2: The Master Join**
It joins the temp table with `UserMasterDataTable` twice:
1.  **Join 1 (C):** To get the Passenger's details (Band, Dept, Designation).
2.  **Join 2 (E):** To get the name of the person who actually *created* the booking (Booked By).

#### **C. Phase 3: The "Union" Logic (One-Way vs. Round-Trip)**
This is the most complex part. Since a **Round Trip** has two legs (Departure and Return), but the UI expects a flat list, the SQL uses `UNION`.
* **Leg 1:** Selects the "From" to "To" journey.
* **Leg 2 (Only for Round Trips):** Selects the "To" back to "From" journey as a separate row so both appear in your React table.



---

### 3. Handling Pagination vs. Export
The procedure checks the `@ExportExcel` bit:
* **If `@ExportExcel = 1`:** It returns **all** records. This is used when you click "Export to Excel" in React.
* **If `@ExportExcel = 0`:** It uses the `OFFSET ... FETCH NEXT` commands.
    * Example: If `@pageNumber = 2` and `@RowsOfPage = 10`, it skips the first 10 rows and gives you rows 11–20.

---

### 4. The Encryption Wrapper
Once the SQL returns the `DataSet`, the BAL converts it to JSON. But notice the Controller return:
`return EncryptedData(_reports.GetDetailedBookingReportPagination(...));`

This means the raw JSON string is passed through an **AES Encryption** algorithm (using the `key` and `iv` you found in your constants file) before it ever leaves the server. This is a security measure to protect Employee PII (Personally Identifiable Information).

### How to debug a value change?
If you see a wrong value in the "Dept" column in your React table:
1.  **Check SQL:** Run the Stored Procedure in SSMS with the same parameters.
2.  **Check UserMasterDataTable:** Verify if that `EmployeeCode` has the correct `Department` assigned.
3.  **Check the Map:** In `DetailedBookingReport.tsx`, ensure the `dataField` matches the column name returned by the SQL (e.g., `Department`).

Would you like to see how to modify the SQL to add a new column to this report?
