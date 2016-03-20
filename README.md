# activflow

[![Build Status](https://travis-ci.org/faxad/activflow.svg?branch=master)](https://travis-ci.org/faxad/activflow)
[![Coverage Status](https://coveralls.io/repos/github/faxad/activflow/badge.svg?branch=master)](https://coveralls.io/github/faxad/activflow?branch=master)
[![Code Health](https://landscape.io/github/faxad/activflow/master/landscape.svg?style=flat)](https://landscape.io/github/faxad/activflow/master)
[![Codacy Badge](https://api.codacy.com/project/badge/grade/82d97392eecb4ffab85403390f6b25af)](https://www.codacy.com/app/fawadhq/activflow)
[![Code Issues](https://www.quantifiedcode.com/api/v1/project/2807a5b5bcdb46258ef0bcf7bb4e4d0f/badge.svg)](https://www.quantifiedcode.com/app/project/2807a5b5bcdb46258ef0bcf7bb4e4d0f)

### Introduction
**ActivFlow** is a generic, light-weight and extensible workflow engine developed to automate complex Business Process operations.
Business flow is mapped as Activities and Transitions

### Technology Stack
- Python 2.7x, 3.4x, 3.5x
- Django 1.9x
- Bootstrap 3.x

### Usage & Configuration

#### Step 1: Workflow App Registration
- A Business Process must be represented in terms of a Django app
- All apps must be registered to **WORKFLOW_APPS** under **core/constants**
```python
WORKFLOW_APPS = ['leave_request']
```

#### Step 2: Activity Configuration
- Activities (States/Nodes) are represented as django models
- Model representing initial Activity, and the following Activities must take **AbstractInitialActivity** and **AbstractActivity** as base respectively
- Custom validation logic must be defined under **clean()** on the activity model
- Custom field specific validation should be defined under **app/validator** and applied to the field as **validators** attribute
```python
from activflow.core.models import AbstractActivity, AbstractInitialActivity
from activflow.leave_request.validators import validate_initial_cap

class RequestInitiation(AbstractInitialActivity):
    """Leave request details"""
    employee_name = CharField("Employee", max_length=200, validators=[validate_initial_cap])
    from = DateField("From Date")
    to = DateField("To Date")
    reason = TextField("Purpose of Leave", blank=True)

    def clean(self):
        """Custom validation logic should go here"""
        pass

class ManagementApproval(AbstractActivity):
    """Management approval"""
    approval_status = CharField(verbose_name="Status", max_length=3, choices=(
        ('APP', 'Approved'), ('REJ', 'Rejected')))
    remarks = TextField("Remarks")

    def clean(self):
        """Custom validation logic should go here"""
        pass

```
#### Step 3: Flow Definition
- A flow is represented by collection of Activities (States/Nodes) connected using Transitions (Edges)
- Rules are applied on transitions to allow transition from one activity to another provided the condition satisfies
- Business Process flow must be defined as **FLOW** under **app/flow**
- As a default behavior, the Role maps OTO with django Group (this can be customized by developers as per the requirements)
```python
from activflow.leave_request.models import RequestInitiation, ManagementApproval
from activflow.leave_request.rules import validate_request

FLOW = {
    'initiate_request': {
        'name': 'Leave Request Initiation',
        'model': RequestInitiation,
        'role': 'Submitter',
        'transitions': {
            'management_approval': validate_request,
        }
    },
    'management_approval': {
        'name': 'Management Approval',
        'model': ManagementApproval,
        'role': 'Approver',
        'transitions': None
    }
}

INITIAL = 'initiate_request'
```
#### Step 4: Business Rules
- Rules and conditions that apply to the transitions must be defined under **app/rules**
```python
def validate_request(self):
    return self.reason == 'Emergency'
```

#### Step 5: Configure Field Visibility
- Include **config.py** in the workflow app and define **ACTIVITY_CONFIG** as Nested Ordered Dictionary
- Define for each activity model the visbility of fields for display on templates and forms 
    - **create:** field will appear on activity create form
    - **update:** field will be available for activity update operation
    - **display:** available for display in activity detail view
```python
from collections import OrderedDict as odict

ACTIVITY_CONFIG = odict([
    ('RequestInitiation', odict([
        ('employee_name', ['create', 'update', 'display']),
        ('from', ['create', 'update', 'display']),
        ('to', ['create', 'update', 'display']),
        ('reason', ['create', 'update', 'display']),
        ('creation_date', ['display']),
        ('last_updated', ['display'])
    ])),
    ('ManagementApproval', odict([
        ('approval_status', ['create', 'update', 'display']),
        ('remarks', ['create', 'update', 'display']),
        ('creation_date', ['display']),
        ('last_updated', ['display'])
    ])),
])

```

#### Step 6: Access/Permission Configuration
**AccessDeniedMixin** under **core/mixins** can be customized to control the access as required
