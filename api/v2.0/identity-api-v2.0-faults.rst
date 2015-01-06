==================================
OpenStack Identity API v2.0 Faults
==================================

When an error occurs, the system returns an HTTP error response code
denoting the type of error. The system also returns additional
information about the fault in the body of the response.

**Example: Identity fault: JSON response**

.. code:: javascript

    {
        "identityFault": {
            "message": "Fault",
            "details": "Error Details...",
            "code": 500
        }
    }


The response body returns the error code for convenience. The message
section returns a human readable message. The details section is
optional and might contain useful information for tracking down an error
(such as, a stack trace).

The root element of the fault (for example, identityFault) might change
depending on the type of error. The following is an example of an
itemNotFound error.

**Example: itemNotFound fault: JSON response**

.. code:: javascript

    {
        "itemNotFound": {
            "message": "Item not found.",
            "details": "Error Details...",
            "code": 404
        }
    }


The following table shows the possible fault types with associated error
codes:

**Table: Fault types**

===================  =====================  ========================
Fault element        Associated error code  Expected in all requests

identityFault        500, 400                           Yes
serviceUnavailable   503                                Yes
badRequest           400                                Yes
unauthorized         401                                Yes
overLimit            413                                No
userDisabled         403                                No
forbidden            403                                No
itemNotFound         404                                No
tenantConflict       409                                No
===================  =====================  ========================
