# Technical Report: Academic Curriculum and Fees Modules

## Executive Summary

This technical report provides an in-depth analysis of two core modules in the Education application built on the Frappe/ERPNext framework:

1. **Academic Curriculum Module** - Program, Course, and Enrollment architecture
2. **Fees (Finance) Module** - Fee structures, schedules, and payment processing

The system is currently designed with a **Program-based enrollment model with optional term support**, which has implications for adaptation to US Higher Education term-based course section enrollment.

---

# Part 1: Academic Curriculum Module

## 1.1 Module Overview

The academic curriculum module manages the complete educational structure including programs, courses, student enrollments, and course scheduling. The architecture follows a hierarchical model:

```
Academic Year
    └── Academic Term (optional)
        └── Program
            └── Program Enrollment
                └── Course Enrollment (created automatically from Program Enrollment)
```

## 1.2 Core DocTypes Architecture

### 1.2.1 Academic Year (`academic_year`)

**Purpose**: Defines the calendar year boundary for academic operations.

**Schema**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `academic_year_name` | Data | Yes | Unique identifier (e.g., "2024-2025") |
| `year_start_date` | Date | Yes | Start of academic year |
| `year_end_date` | Date | Yes | End of academic year |

**Key Relationships**:
- Links to: Student Admission, Student Applicant, Student Group, Student Log, Academic Term, Program Enrollment, Assessment Plan, Assessment Result, Fee Schedule, Fee Structure

**Validation Logic**:
- End date must be after start date

### 1.2.2 Academic Term (`academic_term`)

**Purpose**: Subdivides the academic year into terms (semesters, quarters, trimesters).

**Schema**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `academic_year` | Link | Yes | Parent academic year |
| `term_name` | Data | Yes | Term identifier (e.g., "Fall", "Spring") |
| `term_start_date` | Date | Yes | Term start date |
| `term_end_date` | Date | Yes | Term end date |
| `title` | Data | Auto | Generated as "Academic Year (Term Name)" |

**Key Relationships**:
- Linked by: Student Applicant, Student Group, Student Log, Fee Structure, Fee Schedule, Program Enrollment, Assessment Plan, Assessment Result

**Validation Logic**:
- Term dates must fall within the parent academic year
- No duplicate term names within the same academic year
- Term end date must be after term start date

**Important Note**: Academic Term is **OPTIONAL** in Program Enrollment (not required field), indicating the system was designed with flexibility for non-term-based institutions.

### 1.2.3 Program (`program`)

**Purpose**: Represents an academic degree program or course of study (e.g., "Bachelor of Computer Science").

**Schema**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `program_name` | Data | Yes | Unique program identifier |
| `department` | Link | No | Department offering the program |
| `program_abbreviation` | Data | No | Short code |
| `courses` | Table (Program Course) | No | List of courses in the program |

**Child Table - Program Course**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `course` | Link | Yes | Reference to Course doctype |
| `course_name` | Data | Auto | Fetched from course |
| `required` | Check | Default: 1 | Whether course is mandatory |

**Key Relationships**:
- Linked by: Student Applicant, Program Enrollment, Student Group, Student Log, Assessment Plan, Fee Structure, Fee Schedule, Assessment Result

**Business Logic** (`program.py`):
```python
def get_course_list(self):
    # Returns list of Course documents associated with the program
    program_course_list = self.courses
    course_list = [frappe.get_doc("Course", program_course.course) 
                   for program_course in program_course_list]
    return course_list
```

### 1.2.4 Course (`course`)

**Purpose**: Represents an individual academic course/subject.

**Schema**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `course_name` | Data | Yes | Unique course identifier |
| `department` | Link | No | Department offering the course |
| `description` | Small Text | No | Course description |
| `topics` | Table (Course Topic) | No | Course content breakdown |
| `default_grading_scale` | Link | No | Grading scale for assessments |
| `assessment_criteria` | Table (Course Assessment Criteria) | No | Assessment weightage configuration |

**Important**: The Course doctype does NOT contain:
- Credit hours
- Course sections
- Term/semester association
- Instructor assignment (handled at Course Schedule level)
- Capacity limits

**Validation Logic**:
- Assessment criteria weightages must total 100%

### 1.2.5 Program Enrollment (`program_enrollment`)

**Purpose**: Records a student's enrollment in a program for a specific academic year.

**Schema**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `student` | Link | Yes | Student being enrolled |
| `student_name` | Read Only | Auto | Fetched from student |
| `program` | Link | Yes | Program enrolled in |
| `academic_year` | Link | Yes | Academic year of enrollment |
| `academic_term` | Link | **No** | Optional term specification |
| `enrollment_date` | Date | Yes | Date of enrollment |
| `student_category` | Link | No | Student category |
| `student_batch_name` | Link | No | Batch/cohort identifier |
| `school_house` | Link | No | School house assignment |
| `boarding_student` | Check | No | Hostel resident flag |
| `courses` | Table (Program Enrollment Course) | Auto | Enrolled courses |
| `fees` | Table (Program Fee) | No | Associated fee schedules |

**Naming Convention**: `EDU-ENR-.YYYY.-.#####`

**Workflow State**: Submittable document (Draft → Submitted → Cancelled)

**Key Business Logic** (`program_enrollment.py`):

```python
def validate(self):
    self.set_student_name()
    self.validate_duplication()
    # Auto-populate courses if empty
    if not self.courses:
        self.extend("courses", self.get_courses())

def on_submit(self):
    self.update_student_joining_date()
    self.make_fee_records()          # Creates Sales Invoice/Order for fees
    self.create_course_enrollments() # Creates Course Enrollment records

def get_courses(self):
    # Returns REQUIRED courses from the program
    return frappe.db.sql(
        """select course from `tabProgram Course` 
           where parent = %s and required = 1""",
        (self.program), as_dict=1)

def create_course_enrollments(self):
    for course in self.courses:
        filters = {
            "student": self.student,
            "course": course.course,
            "program_enrollment": self.name,
        }
        if not frappe.db.exists("Course Enrollment", filters):
            filters.update({
                "doctype": "Course Enrollment", 
                "enrollment_date": self.enrollment_date
            })
            frappe.get_doc(filters).save()
```

**Duplication Validation**:
- Prevents duplicate enrollments for the same student + program + academic_year + academic_term combination

### 1.2.6 Course Enrollment (`course_enrollment`)

**Purpose**: Records a student's enrollment in a specific course within a program enrollment.

**Schema**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `program_enrollment` | Link | Yes | Parent program enrollment |
| `student` | Link | Yes | Student enrolled |
| `course` | Link | Yes | Course enrolled in |
| `enrollment_date` | Date | Yes | Enrollment date |
| `student_name` | Data | Auto | Fetched from student |
| `program` | Data | Auto | Fetched from program enrollment |

**Naming Convention**: `EDU-CE-{YYYY}-{#####}`

**Important Characteristics**:
- NOT a submittable document (regular save)
- Automatically created when Program Enrollment is submitted
- Duplication validation: Same student cannot enroll in same course under same program enrollment

**Key Business Logic**:
```python
def validate_duplication(self):
    enrollment = frappe.db.exists("Course Enrollment", {
        "student": self.student,
        "course": self.course,
        "program_enrollment": self.program_enrollment,
        "name": ("!=", self.name),
    })
    if enrollment:
        frappe.throw(_("Student is already enrolled via Course Enrollment {0}"))
```

## 1.3 Student Groups and Scheduling

### 1.3.1 Student Group (`student_group`)

**Purpose**: Groups students for class management, attendance, and course scheduling.

**Schema**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `student_group_name` | Data | Yes | Unique group identifier |
| `academic_year` | Link | Yes | Academic year |
| `academic_term` | Link | No | Optional term |
| `group_based_on` | Select | Yes | "Batch", "Course", or "Activity" |
| `program` | Link | Conditional | Required for Batch/Course groups |
| `batch` | Link | Conditional | Required for Batch groups |
| `course` | Link | Conditional | Required for Course groups |
| `max_strength` | Int | No | Maximum students (0 = no limit) |
| `students` | Table (Student Group Student) | No | List of students |
| `instructors` | Table (Student Group Instructor) | No | Assigned instructors |

**Group Types**:
1. **Batch-based**: Groups all students in a program batch
2. **Course-based**: Groups students enrolled in a specific course
3. **Activity-based**: Ad-hoc groupings for activities

### 1.3.2 Course Schedule (`course_schedule`)

**Purpose**: Schedules individual class sessions.

**Schema**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `student_group` | Link | Yes | Student group for the class |
| `instructor` | Link | Yes | Instructor teaching |
| `course` | Link | Yes | Course being taught |
| `program` | Link | Auto | Fetched from student group |
| `schedule_date` | Date | Yes | Date of class |
| `from_time` | Time | Yes | Start time |
| `to_time` | Time | Yes | End time |
| `room` | Link | Yes | Room/venue |

## 1.4 Data Flow Diagram

```
┌─────────────────┐     ┌─────────────────┐
│ Student         │     │ Program         │
│ Applicant       │────>│                 │
└────────┬────────┘     │  ┌───────────┐  │
         │              │  │ Program   │  │
         │              │  │ Courses   │  │
         ▼              │  └───────────┘  │
┌─────────────────┐     └────────┬────────┘
│ Student         │              │
│                 │              │
└────────┬────────┘              │
         │                       │
         ▼                       ▼
┌────────────────────────────────────────────┐
│              Program Enrollment             │
│  - Links Student + Program + Academic Year  │
│  - Optional Academic Term                   │
│  - Courses (Table: Program Enrollment Course)│
│  - Fees (Table: Program Fee)                │
└─────────────────────┬──────────────────────┘
                      │
          On Submit   │
                      ▼
         ┌────────────┴────────────┐
         │                         │
         ▼                         ▼
┌─────────────────┐    ┌──────────────────────┐
│ Course          │    │ Sales Invoice/Order  │
│ Enrollment      │    │ (for fees)           │
│ (one per course)│    └──────────────────────┘
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Student Group   │
│ (for classes)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Course Schedule │
│ (class sessions)│
└─────────────────┘
```

## 1.5 Enrollment Type Analysis

### Current Model: PROGRAM-BASED Enrollment (with optional Term)

The current system follows a **Program-Based Enrollment** model:

1. **Primary enrollment is to a Program** - Students enroll in programs, not individual courses
2. **Courses are auto-enrolled** - Required program courses are automatically added on program enrollment
3. **Term is optional** - Academic Term is not a required field in Program Enrollment
4. **No course sections** - The Course doctype doesn't support sections/sections
5. **No term-specific course offerings** - Courses are not offered per-term basis

### Key Evidence:

1. **Program Enrollment validation**:
   ```python
   # Duplication check includes academic_term but it's optional
   enrollment = frappe.db.exists("Program Enrollment", {
       "student": self.student,
       "program": self.program,
       "academic_year": self.academic_year,
       "academic_term": self.academic_term,  # Can be null
       "docstatus": ("<", 2),
       "name": ("!=", self.name),
   })
   ```

2. **Course Enrollment has no term field**:
   - Course Enrollment only tracks: program_enrollment, student, course, enrollment_date
   - No academic_term or academic_year directly

3. **Education Settings**:
   - `academic_term_reqd` option exists but defaults to `0` (false)
   - Labeled as "Make Academic Term Mandatory" - confirming it's optional by design

## 1.6 Adaptation Assessment for US Higher Ed Term-Based Model

### Required Changes for US Higher Ed Term-Based Course Section Enrollment:

#### 1. Course Section Concept (NEW DocType Required)

**Current Gap**: No concept of course sections with:
- Different meeting times
- Different instructors
- Capacity limits
- Term-specific offerings

**Recommended New DocType: Course Section**
```
Course Section
├── course (Link to Course)
├── academic_term (Link - Required)
├── section_number (Data, e.g., "001", "002")
├── instructor (Link to Instructor)
├── capacity (Int)
├── enrolled_count (Int, computed)
├── room (Link to Room)
├── meeting_pattern (Table: Meeting Pattern)
│   ├── day_of_week (Select)
│   ├── start_time (Time)
│   └── end_time (Time)
├── credit_hours (Float)
└── status (Select: Open, Closed, Waitlist, Cancelled)
```

#### 2. Term-Based Course Enrollment

**Current Gap**: Course Enrollment links to Program Enrollment, not directly to terms.

**Required Changes**:
1. Add `academic_term` field to Course Enrollment (Required)
2. Add `academic_year` field to Course Enrollment (Required)
3. Add `course_section` field to Course Enrollment (Link to new Course Section)
4. Make Program Enrollment optional or remove the requirement
5. Add `credit_hours_attempted` and `credit_hours_earned` tracking

#### 3. Course Catalog with Term Offerings

**Current Gap**: Courses exist independently of terms.

**Required Changes**:
- Create Term-Course Offering doctype linking courses to specific terms
- Add prerequisites checking logic
- Add co-requisites support

#### 4. Credit Hour System

**Current Gap**: No credit hour tracking.

**Required Additions to Course**:
- `credit_hours` (Int/Float)
- `contact_hours` (Int)
- `lab_hours` (Int, optional)

**Required Additions to Student/Program Enrollment**:
- Cumulative GPA calculation
- Credit hours attempted/earned tracking
- Academic standing logic

#### 5. Registration Period Management

**Current Gap**: No concept of registration windows.

**Required New DocType: Registration Period**
```
Registration Period
├── academic_term (Link)
├── start_datetime (Datetime)
├── end_datetime (Datetime)
├── student_groups (Table - who can register)
├── add_drop_deadline (Date)
└── withdrawal_deadline (Date)
```

#### 6. Prerequisites and Course Requirements

**Current Gap**: No prerequisites system.

**Required Changes**:
- Add prerequisites table to Course doctype
- Add grade requirement field for prerequisites
- Implement validation logic during enrollment

### Effort Estimation:

| Component | Effort Level | Description |
|-----------|--------------|-------------|
| Course Section DocType | High | New core doctype with scheduling |
| Term-Based Course Enrollment | Medium | Modify existing Course Enrollment |
| Credit Hour System | Medium | Add fields and calculation logic |
| Prerequisites System | Medium | New validation logic |
| Registration Management | High | New business process |
| Waitlist Management | Medium | New feature |
| GPA Calculation | Medium | New calculation engine |
| Drop/Add/Withdraw Process | Medium | New workflow |

### Compatibility Considerations:

1. **Breaking Changes**: Converting to term-based enrollment would be a fundamental architecture change
2. **Data Migration**: Existing Program Enrollments would need to be mapped to the new structure
3. **Integration Points**: Fee calculation, attendance, and grading would need updates
4. **Frontend Updates**: Significant UI changes for registration workflow

---

# Part 2: Fees (Finance) Module

## 2.1 Module Overview

The Fees module handles all financial aspects of student education including fee structures, scheduling, invoicing, and payment processing. It is tightly integrated with ERPNext's accounting system.

## 2.2 Core DocTypes Architecture

### 2.2.1 Fee Category (`fee_category`)

**Purpose**: Defines types of fees that can be charged (e.g., "Tuition", "Lab Fee", "Library Fee").

**Schema**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `category_name` | Data | Yes | Unique fee category name |
| `description` | Small Text | No | Category description |
| `item` | Link | Read Only | Auto-created Item for billing |
| `item_defaults` | Table (Fee Category Default) | No | Accounting defaults per company |

**Child Table - Fee Category Default**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `company` | Link | Yes | Company for multi-company setups |
| `income_account` | Link | No | Default income account |
| `selling_cost_center` | Link | No | Default cost center |

**Key Business Logic**:
```python
def after_insert(self):
    # Automatically create an Item for this fee category
    item_name = create_item(self)
    self.item = item_name
    self.save()

def on_trash(self):
    # Delete associated item when fee category is deleted
    frappe.delete_doc("Item", self.name, force=1)
```

**Item Creation**:
- Creates an Item with `item_group = "Fee Component"`
- Item is a service item (non-stock)
- Income account and cost center inherited from Fee Category Default

### 2.2.2 Fee Structure (`fee_structure`)

**Purpose**: Defines the fee breakdown for a specific program, academic year, and optionally term.

**Schema**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `program` | Link | Yes | Program this fee applies to |
| `academic_year` | Link | Yes | Academic year |
| `academic_term` | Link | No | Optional term specification |
| `student_category` | Link | No | Optional student category filter |
| `components` | Table (Fee Component) | Yes | Fee breakdown |
| `total_amount` | Currency | Read Only | Calculated total |
| `receivable_account` | Link | Yes | AR account for receivables |
| `cost_center` | Link | No | Cost center for accounting |
| `company` | Link | No | Company for ledger entries |

**Child Table - Fee Component**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `fees_category` | Link | Yes | Link to Fee Category |
| `description` | Small Text | Auto | Fetched from fee category |
| `amount` | Currency | Yes | Base amount |
| `item` | Link | Read Only | Fetched from fee category |
| `discount` | Percent | Default: 0 | Discount percentage |
| `total` | Float | Computed | Amount after discount |

**Workflow State**: Submittable document

**Key Business Logic**:
```python
def calculate_total(self):
    """Calculates total amount."""
    self.total_amount = 0
    for d in self.components:
        d.total = flt(d.amount) - (d.amount * (flt(d.discount) / 100))
        self.total_amount += d.total

def before_submit(self):
    for component in self.components:
        # Create item for each component if it doesn't exist
        if not component.get("item"):
            self.create_item_master_and_save_item_in_fee_component(component)
```

**Fee Plan Distribution** (`get_amount_distribution_based_on_fee_plan`):
Supports splitting fees into:
- Monthly (12 installments)
- Quarterly (4 installments)
- Semi-Annually (2 installments)
- Term-Wise (based on academic terms in the year)
- Annually (single payment)

### 2.2.3 Fee Schedule (`fee_schedule`)

**Purpose**: Schedules fee collection for a group of students with specific due dates.

**Schema**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `fee_structure` | Link | Yes | Parent fee structure |
| `posting_date` | Date | Yes | Invoice posting date |
| `due_date` | Date | Yes | Payment due date |
| `academic_year` | Link | Yes | Academic year |
| `academic_term` | Link | No | Optional term |
| `student_category` | Link | Read Only | From fee structure |
| `program` | Link | Read Only | From fee structure |
| `student_groups` | Table (Fee Schedule Student Group) | Yes | Target student groups |
| `components` | Table (Fee Component) | Yes | Fee components |
| `total_amount` | Currency | Read Only | Per-student amount |
| `grand_total` | Currency | Read Only | Total for all students |
| `send_email` | Check | No | Send payment request email |
| `status` | Select | Read Only | Draft/Invoice Pending/Order Pending/In Process/Invoice Created/Order Created/Failed |

**Statuses**:
- `Draft` - Not submitted
- `Invoice Pending` - Submitted, awaiting invoice creation (if create_so=False)
- `Order Pending` - Submitted, awaiting order creation (if create_so=True)
- `In Process` - Invoice/Order creation in progress
- `Invoice Created` - All invoices created
- `Order Created` - All orders created
- `Failed` - Error during creation

**Key Business Logic**:
```python
def create_fees(self):
    """Triggers fee record creation for all students in the schedule."""
    self.db_set("status", "In Process")
    total_records = sum([int(d.total_students) for d in self.student_groups])
    
    if total_records > 10:
        # Background job for large batches
        enqueue(generate_fees, queue="default", timeout=6000, 
                event="generate_fees", fee_schedule=self.name)
    else:
        generate_fees(self.name)
```

**Fee Generation Process**:
```python
def generate_fees(fee_schedule):
    doc = frappe.get_doc("Fee Schedule", fee_schedule)
    create_so = frappe.db.get_single_value("Education Settings", "create_so")
    
    for d in doc.student_groups:
        students = get_students(d.student_group, doc.academic_year, 
                               doc.academic_term, doc.student_category)
        for student in students:
            if create_so:
                create_sales_order(fee_schedule, student.student)
            else:
                create_sales_invoice(fee_schedule, student.student)
```

### 2.2.4 Fees (`fees`) - Legacy DocType

**Purpose**: Original fee document (largely replaced by Sales Invoice integration).

**Schema**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `student` | Link | Yes | Student being charged |
| `program_enrollment` | Link | Yes | Related program enrollment |
| `fee_structure` | Link | Yes | Fee structure used |
| `fee_schedule` | Link | Read Only | Associated fee schedule |
| `components` | Table (Fee Component) | Yes | Fee breakdown |
| `grand_total` | Currency | Read Only | Total amount |
| `outstanding_amount` | Currency | Read Only | Remaining balance |
| `posting_date` | Date | Yes | Invoice date |
| `due_date` | Date | Yes | Payment due date |
| `receivable_account` | Link | Yes | AR account |
| `income_account` | Link | No | Revenue account |
| `cost_center` | Link | No | Cost center |

**Important**: This doctype creates GL entries directly (not through Sales Invoice).

**GL Entry Logic**:
```python
def make_gl_entries(self):
    if not self.grand_total:
        return
    
    # Debit: Student Receivable
    student_gl_entries = self.get_gl_dict({
        "account": self.receivable_account,
        "party_type": "Student",
        "party": self.student,
        "against": self.income_account,
        "debit": self.grand_total,
        "against_voucher": self.name,
        "against_voucher_type": self.doctype,
    })
    
    # Credit: Income Account
    fee_gl_entry = self.get_gl_dict({
        "account": self.income_account,
        "against": self.student,
        "credit": self.grand_total,
        "cost_center": self.cost_center,
    })
    
    make_gl_entries([student_gl_entries, fee_gl_entry])
```

### 2.2.5 Program Fee (Child Table)

**Purpose**: Links fee schedules to program enrollments.

**Schema**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `fee_schedule` | Link | Yes | Fee Schedule reference |
| `academic_term` | Link | Auto | Fetched from fee schedule |
| `student_category` | Link | Auto | Fetched from fee schedule |
| `due_date` | Date | Auto | Fetched from fee schedule |
| `amount` | Currency | Auto | Fetched from fee schedule |

## 2.3 Fee Processing Workflow

### 2.3.1 Configuration Flow

```
1. Create Fee Categories
   └── "Tuition Fee", "Lab Fee", "Library Fee", etc.
   └── Auto-creates Items in "Fee Component" item group
   └── Set accounting defaults per company

2. Create Fee Structure
   └── Select Program + Academic Year + (Optional) Term
   └── Add Fee Components with amounts
   └── Set accounting accounts
   └── Submit to finalize

3. Create Fee Schedule(s) from Fee Structure
   └── Set posting date and due date
   └── Select student groups
   └── Submit to enable fee generation
   └── Click "Create Fees" to generate invoices/orders
```

### 2.3.2 Fee Generation Flow

```
┌─────────────────┐
│  Fee Structure  │
│  (Template)     │
└────────┬────────┘
         │ Create Schedule
         ▼
┌─────────────────┐
│  Fee Schedule   │
│  (Per Due Date) │
└────────┬────────┘
         │ Submit & Create Fees
         ▼
┌─────────────────────────────────────────────┐
│  For each student in student_groups:        │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │ Education Settings.create_so = True  │   │
│  │                ↓                     │   │
│  │     Create Sales Order               │   │
│  │     (Customer = Student.customer)    │   │
│  │                ↓                     │   │
│  │     Convert to Sales Invoice         │   │
│  │     (Manual or Auto)                 │   │
│  │                ↓                     │   │
│  │     Create Payment Entry             │   │
│  └──────────────────────────────────────┘   │
│                   OR                         │
│  ┌──────────────────────────────────────┐   │
│  │ Education Settings.create_so = False │   │
│  │                ↓                     │   │
│  │     Create Sales Invoice             │   │
│  │     (Customer = Student.customer)    │   │
│  │                ↓                     │   │
│  │     Create Payment Entry             │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### 2.3.3 Program Enrollment Fee Trigger

When a Program Enrollment is submitted, if it has fees configured:

```python
def make_fee_records(self):
    create_so = frappe.db.get_single_value("Education Settings", "create_so")
    
    for d in self.fees:  # Program Fee child table
        if create_so:
            sales_order = create_sales_order(d.fee_schedule, self.student)
        else:
            sales_invoice = create_sales_invoice(d.fee_schedule, self.student)
```

## 2.4 Integration Points

### 2.4.1 Student-Customer Relationship

Every Student automatically creates/links to a Customer:

```python
class Student(Document):
    def on_update(self):
        self.set_missing_customer_details()
    
    def create_customer(self):
        customer = frappe.get_doc({
            "doctype": "Customer",
            "customer_name": self.student_name,
            "customer_group": self.customer_group or "Student",
            "customer_type": "Individual",
        }).insert()
        frappe.db.set_value("Student", self.name, "customer", customer.name)
```

### 2.4.2 Sales Invoice Integration

Fee Schedule creates Sales Invoices with:
- `customer` = Student's linked Customer
- `student` = Student reference (custom field)
- `fee_schedule` = Fee Schedule reference (custom field)
- Items from Fee Components

```python
def get_fees_mapped_doc(fee_schedule, doctype, student_id, customer):
    table_map = {
        "Fee Schedule": {
            "doctype": doctype,
            "field_map": {
                "name": "fee_schedule",
                "due_date": "due_date",
                "posting_date": "posting_date",
            },
        },
        "Fee Component": {
            "doctype": "Sales Invoice Item",
            "field_map": {
                "item": "item_code",
                "amount": "price_list_rate",
                "discount": "discount_percentage",
            },
        },
    }
    doc = get_mapped_doc("Fee Schedule", fee_schedule, table_map)
    doc.student = student_id
    doc.customer = customer
    return doc
```

### 2.4.3 Payment Processing (Razorpay Integration)

The billing.py module provides Razorpay payment gateway integration:

```python
@frappe.whitelist()
def get_payment_options(doctype, docname, phone, currency=None):
    """Create Razorpay order for online payment."""
    details = get_details(docname)  # Sales Invoice details
    client = get_client()  # Razorpay client
    order = create_order(client, details.outstanding_amount, details.currency)
    
    return {
        "key_id": frappe.db.get_single_value("Education Settings", "razorpay_key"),
        "order_id": order["id"],
        "amount": order["amount"],
        ...
    }

@frappe.whitelist()
def handle_payment_success(response, against_invoice, billing_details):
    """Process successful payment and create Payment Entry."""
    client.utility.verify_payment_signature(response)
    
    pe = get_payment_entry("Sales Invoice", against_invoice)
    pe.reference_no = response["razorpay_order_id"]
    pe.reference_date = nowdate()
    pe.save(ignore_permissions=True)
    pe.submit()
```

### 2.4.4 Accounting Integration

**GL Entry Flow**:
```
Sales Invoice Submission
        │
        ├──► Debit: Receivable Account (Student as Party)
        │    [Increases Student Receivable]
        │
        └──► Credit: Income Account
             [Records Fee Revenue]

Payment Entry Submission
        │
        ├──► Debit: Bank/Cash Account
        │    [Records payment receipt]
        │
        └──► Credit: Receivable Account (Student as Party)
             [Reduces Student Receivable]
```

### 2.4.5 Module Connections

```
┌─────────────────────────────────────────────────────────────────┐
│                        FEES MODULE                               │
└─────────────────────────────────────────────────────────────────┘
          │                     │                    │
          ▼                     ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐
│    EDUCATION    │  │    ACCOUNTS     │  │      SELLING        │
│    MODULE       │  │    MODULE       │  │      MODULE         │
├─────────────────┤  ├─────────────────┤  ├─────────────────────┤
│ • Student       │  │ • GL Entry      │  │ • Customer          │
│ • Program       │  │ • Payment Entry │  │ • Sales Invoice     │
│ • Program       │  │ • Account       │  │ • Sales Order       │
│   Enrollment    │  │ • Cost Center   │  │ • Item              │
│ • Student Group │  │ • Payment       │  │ • Payment Request   │
│ • Academic Year │  │   Request       │  │                     │
│ • Academic Term │  │                 │  │                     │
└─────────────────┘  └─────────────────┘  └─────────────────────┘
```

## 2.5 Education Settings (Fee-Related)

| Setting | Type | Description |
|---------|------|-------------|
| `create_so` | Check | If enabled, creates Sales Order instead of Sales Invoice |
| `auto_submit_sales_invoice` | Check | Auto-submit Sales Invoice on creation |
| `auto_submit_sales_order` | Check | Auto-submit Sales Order on creation |
| `sales_invoice_posting_date_fee_schedule` | Check | Use Fee Schedule's posting date for SI |
| `sales_order_transaction_date_fee_schedule` | Check | Use Fee Schedule's posting date for SO |
| `razorpay_key` | Data | Razorpay API Key |
| `razorpay_secret` | Password | Razorpay API Secret |

## 2.6 Reports

### 2.6.1 Program Wise Fee Collection

Reports on fees collected and outstanding by program:

```python
def get_data(filters=None):
    # From Sales Invoice
    sales_invoice_details = (
        frappe.qb.from_(sales_invoice)
        .inner_join(fee_schedule)
        .on(sales_invoice.fee_schedule == fee_schedule.name)
        .select(
            fee_schedule.program,
            (Sum(sales_invoice.grand_total) - Sum(sales_invoice.outstanding_amount)).as_("paid_amount"),
            Sum(sales_invoice.outstanding_amount).as_("outstanding_amount"),
            Sum(sales_invoice.grand_total).as_("grand_total"),
        )
        .where(sales_invoice.docstatus == 1)
        .groupby(fee_schedule.program)
    ).run(as_dict=1)
```

## 2.7 Frontend Integration

The Vue.js frontend (`frontend/src/pages/Fees.vue`) displays:
- Student invoices with status (Paid, Unpaid, Overdue, Partly Paid)
- Payment dates and due dates
- Download invoice functionality
- Pay Now button triggering Razorpay payment dialog

```javascript
const feesResource = createResource({
    url: 'education.education.api.get_student_invoices',
    params: { student: studentInfo.name },
    onSuccess: (response) => {
        tableData.rows = response.invoices
    },
    auto: true,
})
```

## 2.8 Recommendations for Enhancement

### 2.8.1 Term-Based Fee Improvements

1. **Per-Term Fee Structures**: Create separate fee structures for each term instead of annual structures
2. **Automatic Term Detection**: When enrolling in a term, auto-apply the term's fee structure
3. **Prorated Fees**: Support for calculating prorated fees for mid-term enrollments

### 2.8.2 Payment Plan Support

1. **Installment Plans**: Allow defining custom payment schedules beyond the fixed Monthly/Quarterly options
2. **Late Fee Calculation**: Automatic late fee assessment based on due date
3. **Financial Aid Integration**: Deduction of scholarships/grants from fee totals

### 2.8.3 Better Reporting

1. **Student Account Statement**: Complete financial history per student
2. **Aging Report**: Outstanding balances by age buckets
3. **Revenue Recognition**: Defer revenue recognition for multi-term payments

---

# Conclusion

## Summary

The Education module is built as a **Program-centric system** with optional term support, primarily designed for K-12 or similar educational institutions where students enroll in programs and automatically receive all program courses.

### Curriculum Module
- **Strength**: Clean hierarchy from Program → Enrollment → Courses
- **Limitation**: No course sections, no prerequisites, no credit hours
- **Adaptability for US Higher Ed**: Requires significant structural changes

### Fees Module
- **Strength**: Robust integration with ERPNext accounting, flexible fee structures
- **Strength**: Supports multiple payment methods including online payment
- **Limitation**: Tightly coupled to program enrollment, not course-level billing
- **Adaptability for US Higher Ed**: Would need course-level fee support

## Adaptation Effort for US Higher Ed

| Requirement | Current Support | Effort to Add |
|-------------|-----------------|---------------|
| Term-based enrollment | Partial (optional term) | Medium |
| Course sections | Not supported | High |
| Credit hours | Not supported | Medium |
| Prerequisites | Not supported | Medium |
| Per-course fees | Not supported | High |
| Registration windows | Not supported | Medium |
| Waitlists | Not supported | Medium |
| GPA calculation | Not supported | Medium |
| Transcript generation | Not supported | High |

**Overall Assessment**: The current architecture would require substantial modification to support US Higher Education term-based, course-section enrollment model. A better approach might be to create new dedicated doctypes for Higher Ed while keeping the existing ones for K-12 scenarios.
