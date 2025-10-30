# Auxiliary Files Support in OBOT Jobs API

## Overview

The Jobs API now supports auxiliary files (aux_files) that are associated with casefiles. Auxiliary files are input files required by OLGA simulations, such as:

- PVT tables (`.tab` files)
- Trend files (`.tpl` files)
- Custom fluid property files
- Boundary condition files
- Any other input files referenced by the casefile

## Data Model

```
Job
  └── Kase
       └── Casefile
            └── CasefileAssociation
                 └── AuxFile
```

- **AuxFile**: Represents a physical file on the shared storage
- **CasefileAssociation**: Join table linking casefiles to their auxiliary files
- **Automatic flags**: `is_output` and `is_downloaded` are automatically set to `false` for input files

## API Request Format

When creating a job, include `aux_files` as a nested array under each casefile:

```json
{
  "job": {
    "description": "Production Run",
    "project_name": "North Sea Pipeline",
    "olga_version": "7.3.5",
    "priority_id": 1,
    "engine_id": 1
  },
  "casefiles": [
    {
      "file_name": "base_case.olga",
      "path": "/shared/projects/north_sea/base_case.olga",
      "modification_time": "2025-10-07T15:30:00Z",
      "status": 1,
      "modules": ["Pipeline", "Multiphase", "Compositional"],
      "aux_files": [
        {
          "file_name": "pvt_table.tab",
          "path": "/shared/projects/north_sea/pvt_table.tab",
          "modification_time": "2025-10-07T15:00:00Z"
        },
        {
          "file_name": "boundary_conditions.tpl",
          "path": "/shared/projects/north_sea/boundary_conditions.tpl",
          "modification_time": "2025-10-07T14:30:00Z"
        }
      ]
    }
  ]
}
```

## Required Fields for Aux Files

- `file_name`: Name of the auxiliary file
- `path`: Full path to the file on shared storage
- `modification_time`: (Optional) Timestamp of when the file was last modified

## API Response Format

When retrieving job details, aux_files are included in the casefile object:

```json
{
  "id": 123,
  "description": "Production Run",
  "kases": [
    {
      "id": 456,
      "casefile": {
        "id": 789,
        "file_name": "base_case.olga",
        "path": "/shared/projects/north_sea/base_case.olga",
        "status": 1,
        "aux_files": [
          {
            "id": 101,
            "file_name": "pvt_table.tab",
            "path": "/shared/projects/north_sea/pvt_table.tab",
            "modification_time": "2025-10-07T15:00:00Z",
            "is_output": false,
            "is_downloaded": false
          },
          {
            "id": 102,
            "file_name": "boundary_conditions.tpl",
            "path": "/shared/projects/north_sea/boundary_conditions.tpl",
            "modification_time": "2025-10-07T14:30:00Z",
            "is_output": false,
            "is_downloaded": false
          }
        ]
      }
    }
  ]
}
```

## Behavior

### Deduplication
- If an aux_file with the same `path` and `file_name` already exists, it will be reused
- If a casefile-auxfile association already exists, it won't be duplicated

### Input Files vs Output Files
- Aux files created via API are always **input files** (`is_output = false`)
- Output files are created by a different service after simulation completion
- The API only handles input files during job creation

### Transaction Safety
- All aux_file creation happens within the same transaction as job creation
- If any aux_file creation fails, the entire job creation is rolled back

## Example Use Cases

### 1. Simple Case with One Aux File
```json
{
  "casefiles": [
    {
      "file_name": "pipeline.olga",
      "path": "/shared/cases/pipeline.olga",
      "modules": ["Pipeline"],
      "aux_files": [
        {
          "file_name": "fluid.tab",
          "path": "/shared/cases/fluid.tab"
        }
      ]
    }
  ]
}
```

### 2. Multiple Casefiles with Shared Aux File
```json
{
  "casefiles": [
    {
      "file_name": "case1.olga",
      "path": "/shared/cases/case1.olga",
      "modules": ["Pipeline"],
      "aux_files": [
        {
          "file_name": "common_pvt.tab",
          "path": "/shared/data/common_pvt.tab"
        }
      ]
    },
    {
      "file_name": "case2.olga",
      "path": "/shared/cases/case2.olga",
      "modules": ["Pipeline"],
      "aux_files": [
        {
          "file_name": "common_pvt.tab",
          "path": "/shared/data/common_pvt.tab"
        }
      ]
    }
  ]
}
```
Note: The `common_pvt.tab` file will only be created once and associated with both casefiles.

### 3. Case with Multiple Aux Files
```json
{
  "casefiles": [
    {
      "file_name": "complex_case.olga",
      "path": "/shared/cases/complex_case.olga",
      "modules": ["Pipeline", "Multiphase", "Compositional"],
      "aux_files": [
        {
          "file_name": "pvt_data.tab",
          "path": "/shared/data/pvt_data.tab"
        },
        {
          "file_name": "inlet_trend.tpl",
          "path": "/shared/data/inlet_trend.tpl"
        },
        {
          "file_name": "outlet_pressure.tpl",
          "path": "/shared/data/outlet_pressure.tpl"
        },
        {
          "file_name": "ambient_temp.tpl",
          "path": "/shared/data/ambient_temp.tpl"
        }
      ]
    }
  ]
}
```

## Error Handling

If aux_file creation fails, you'll receive an error message:

```json
{
  "error": "Failed to create aux_file: [error details]"
}
```

Common reasons for failure:
- Invalid path or file_name
- Database constraint violations
- Transaction rollback due to other job creation failures

## Migration Notes

For existing codebases:
- The `aux_files` array is optional - if omitted, no aux files are created
- Existing jobs without aux files continue to work normally
- This feature is backward compatible with the previous API version
