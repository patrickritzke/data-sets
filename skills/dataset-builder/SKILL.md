---
name: dataset-builder
description: Build IntApp Wilco datasets (QueryIntegrations). Use when the user wants to create, modify, or understand a dataset/query integration in Wilco. Covers both REST (OpenLink) and DSL query types.
---

# Dataset Builder Skill for IntApp Wilco

Build `QueryIntegration` definitions for the IntApp Wilco `.ixt` export format. Datasets are the query engine behind dropdowns, calculated values, and data tables in Wilco forms and workflows.

## Overview

Every dataset is a `QueryIntegration` with these top-level fields:

| Field | Description | Common Values |
|-------|-------------|---------------|
| `Name` | Display name for the dataset | Any descriptive string |
| `Active` | Whether the dataset is live | `true` / `false` |
| `Draft` | Unpublished draft | `false` |
| `DatasourceId` | Which datasource to use | `1` (OpenLink/primary) |
| `QueryIntegrationType` | Shape of the result | `KeyValue`, `Value`, `Datatable` |
| `QueryIntegrationObjectType` | Object binding | `None` (default) |
| `IgnoreClientMatterSecurity` | Bypass C/M security | `false` (default) |
| `useDeletedFlagForParties` | Include deleted parties | `false` for REST, `true` for DSL |
| `SchemaVersion` | Schema version | `4300` |

### Result Types

- **`KeyValue`** — Two-column output (Key + Value) used for dropdown/lookup fields
- **`Value`** — Single scalar value used for computed/display fields
- **`Datatable`** — Multi-column grid output used for repeating tables

---

## Two Query Modes

### Mode 1: REST Query (OpenLink API)

Used to call built-in Wilco REST endpoints. Set `RunAsDSLQuery: false` and populate `DataSetSettings`.

```xml
<QueryBuilder>
  <DataSetSettings>
    <CollectionType>All</CollectionType>
    <DataSourceId>lookup</DataSourceId>
    <OperationId>get-offices</OperationId>
    <FieldNames>
      <string>[officeCode]</string>
      <string>[office]</string>
    </FieldNames>
    <ParameterExpressions>
      <RestParameterExpression>
        <Operator>Equals</Operator>
        <Parameter>
          <IsExternal>false</IsExternal>
          <IsRequired>false</IsRequired>
          <Name>onlyActive</Name>
        </Parameter>
        <Value>true</Value>
      </RestParameterExpression>
    </ParameterExpressions>
    <ReturnDistinctRows>false</ReturnDistinctRows>
  </DataSetSettings>
  <RunAsDSLQuery>false</RunAsDSLQuery>
</QueryBuilder>
```

**Available DataSources and Operations:**

| DataSource | Operations |
|------------|-----------|
| `lookup` | `get-offices`, `get-countries`, `get-currencies`, `get-departments`, `get-industries`, `get-address-types` |
| `matter` | `find-matters`, `get-matter`, `get-matter-statuses-without-current`, `get-financial-info`, `find-addresses` |
| `client` | `find-clients`, `get-client`, `get-financial-info`, `get-matters`, `find-addresses` |
| `user` | `find-user-groups`, `find-subordinates-by-user`, `find-matter-users`, `find-client-users` |
| `related-parties` | `find-parties`, `find-client-parties`, `find-matter-parties` |
| `conflicts-search` | `get-affiliation-types`, `get-importance-levels`, `get-relationship-types` |
| `terms` | `get-terms`, `get-terms-entity-profile-link` |
| `dateTime` | `get-now` |
| `virtualtables` | Virtual table name as OperationId |

**Parameter types:**
- `IsExternal: false` — static/hardcoded value (put the value in `<Value>`)
- `IsExternal: true` — bound to a runtime search term or form field (leave `<Value>` empty/nil)

---

### Mode 2: DSL Query

Used for SQL-like queries against Wilco's internal data model. Set `RunAsDSLQuery: true` and populate `DslQuerySettings`.

```xml
<QueryBuilder>
  <DataSetSettings>
    <!-- all fields nil/empty for DSL queries -->
    <ReturnDistinctRows>false</ReturnDistinctRows>
  </DataSetSettings>
  <DslQuerySettings>
    <Distinct>false</Distinct>
    <EnforceSecurity>true</EnforceSecurity>
    <QueryLimit>500</QueryLimit>
    <Source>
      <Alias>Parties</Alias>
      <TableId>Parties</TableId>
    </Source>
    <SelectFields>
      <SelectExpression>
        <ColumnName>Key</ColumnName>
        <Expression>[#:#system#:#Parties#:#Parties#:#Id#:#]</Expression>
      </SelectExpression>
      <SelectExpression>
        <ColumnName>Value</ColumnName>
        <Expression>[#:#system#:#Parties#:#Parties#:#Name#:#]</Expression>
      </SelectExpression>
    </SelectFields>
  </DslQuerySettings>
  <RunAsDSLQuery>true</RunAsDSLQuery>
</QueryBuilder>
```

---

## DSL Query Reference

### Field Reference Syntax

All field references in DSL expressions use this token format:

```
#:#system#:#<TableAlias>#:#<TableId>#:#<FieldName>#:#
```

Examples:
- `[#:#system#:#Parties#:#Parties#:#Name#:#]`
- `[#:#system#:#Clients#:#Clients#:#ClientId#:#]`
- `[#:#system#:#Activities#:#Activities#:#CreatedOn#:#]`

For **virtual tables**:
```
#:#virtual#:#<VirtualTableName>#:#<VirtualTableName>#:#<FieldName>#:#<DataType>#:#
```

For **external/form fields** (used in ExternalParameters):
```
#:#external#:#<guid>#:#<DataType>#:#<FieldName>#:#
```

### Source

The primary table for the query. Use `TableId` for system tables, `OperationId` for virtual tables.

```xml
<!-- System table -->
<Source>
  <Alias>Clients</Alias>
  <TableId>Clients</TableId>
</Source>

<!-- Virtual table -->
<Source>
  <Alias>My Virtual Table</Alias>
  <OperationId>My Virtual Table</OperationId>
</Source>
```

### Joins

```xml
<Joined>
  <JoinedTable>
    <JoinType>Outer</JoinType>  <!-- null = inner join, "Outer" = left outer -->
    <Table>
      <Alias>Activities</Alias>
      <TableId>Activities</TableId>
    </Table>
    <JoinCondition>
      <Left><Value>#:#system#:#Activities#:#Activities#:#ComponentId#:#</Value></Left>
      <Operator>Equals</Operator>
      <Right><Value>#:#system#:#Clients#:#Clients#:#ComponentIdFk#:#</Value></Right>
    </JoinCondition>
  </JoinedTable>
  <!-- additional JoinedTable elements as needed -->
</Joined>
```

**Compound join condition** (AND two conditions):
```xml
<JoinCondition>
  <Left>
    <Left><Value>#:#system#:#TableA#:#TableA#:#FieldA#:#</Value></Left>
    <Operator>Equals</Operator>
    <Right><Value>#:#system#:#TableB#:#TableB#:#FieldB#:#</Value></Right>
  </Left>
  <Operator>And</Operator>
  <Right>
    <Left><Value>#:#system#:#TableA#:#TableA#:#FieldC#:#</Value></Left>
    <Operator>Equals</Operator>
    <Right><Value>#:#system#:#TableB#:#TableB#:#FieldD#:#</Value></Right>
  </Right>
</JoinCondition>
```

### Conditions (WHERE)

```xml
<Conditions>
  <CollectionType>All</CollectionType>  <!-- All = AND, Any = OR -->
  <Conditions>
    <DataSetFilterCondition>
      <OrdinalIdx>1</OrdinalIdx>
      <Operator>Equals</Operator>
      <Left><Value>#:#system#:#Clients#:#Clients#:#ClientId#:#</Value></Left>
      <Right>
        <!-- Static value -->
        <DefaultValue>ABC123</DefaultValue>
        <External>false</External>
        <Name>p0</Name>
        <ValueType>String</ValueType>
      </Right>
    </DataSetFilterCondition>
  </Conditions>
</Conditions>
```

**External (runtime) parameter** in condition Right:
```xml
<Right>
  <DefaultValue i:nil="true"/>
  <External>true</External>
  <Name>p0</Name>
  <ValueType>String</ValueType>  <!-- String, Dictionary, Number, Boolean -->
</Right>
```

**Condition Operators:** `Equals`, `NotEquals`, `In`, `NotIn`, `Contains`, `StartsWith`, `IsNull`, `IsNotNull`, `GreaterThan`, `LessThan`

Use `In` with `ValueType: Dictionary` for multi-value parameters.

### Select Fields

For **KeyValue** results, use column names `Key` and `Value`:
```xml
<SelectFields>
  <SelectExpression>
    <ColumnName>Key</ColumnName>
    <Expression>[#:#system#:#Parties#:#Parties#:#Id#:#]</Expression>
  </SelectExpression>
  <SelectExpression>
    <ColumnName>Value</ColumnName>
    <Expression>[#:#system#:#Parties#:#Parties#:#Name#:#] + ' (' + [#:#system#:#Clients#:#Clients#:#ClientId#:#] + ')'</Expression>
  </SelectExpression>
</SelectFields>
```

For **Value** results, use a single column named `Value`:
```xml
<SelectFields>
  <SelectExpression>
    <ColumnName>Value</ColumnName>
    <Expression>Round([#:#external#:#guid1#:#Number#:#Fees#:#] + [#:#external#:#guid2#:#Number#:#Costs#:#], 0)</Expression>
  </SelectExpression>
</SelectFields>
```

For **Datatable** results, use any descriptive column names:
```xml
<SelectFields>
  <SelectExpression>
    <ColumnName>Client Id</ColumnName>
    <Expression>[#:#system#:#Clients#:#Clients#:#ClientId#:#]</Expression>
  </SelectExpression>
  <SelectExpression>
    <ColumnName>Client Name</ColumnName>
    <Expression>[#:#system#:#Clients#:#Clients#:#Name#:#]</Expression>
  </SelectExpression>
</SelectFields>
```

**Expression SQL functions:** `Round()`, `case when ... then ... else ... end`, string concatenation with `+`, `isnull()`, `convert()`, `dateadd()`, `datediff()`

### Sort Fields

```xml
<SortFields>
  <SortField>
    <Direction>ASCENDING</Direction>  <!-- ASCENDING or DESCENDING -->
    <Field><Value>#:#system#:#Clients#:#Clients#:#ClientId#:#</Value></Field>
  </SortField>
</SortFields>
```

### External Parameters (form field references)

When a DSL query reads values from form questions, declare them in `ExternalParameters`:

```xml
<ExternalParameters>
  <EncodedExternalParameter>
    <Value>#:#external#:#<guid>#:#<DataType>#:#<FieldName>#:#</Value>
  </EncodedExternalParameter>
</ExternalParameters>
```

---

## Complete XML Template

```xml
<QueryIntegration>
  <Id><!-- assign new unique int --></Id>
  <SchemaVersion>4300</SchemaVersion>
  <Active>true</Active>
  <DatasourceId>1</DatasourceId>
  <Draft>false</Draft>
  <IgnoreClientMatterSecurity>false</IgnoreClientMatterSecurity>
  <IsIntegrationTypeUserSelected>false</IsIntegrationTypeUserSelected>
  <Name>My New Dataset</Name>
  <QueryBuilder>
    <DataSetSettings>
      <AdvancedExpression i:nil="true"/>
      <AggregationFunction i:nil="true"/>
      <CollectionType>All</CollectionType>
      <ComputedFieldFlags i:nil="true"/>
      <DataSourceId i:nil="true"/>
      <ExternalParameters/>
      <FieldColumnTitles i:nil="true"/>
      <FieldNames i:nil="true"/>
      <OperationId i:nil="true"/>
      <ParameterExpressions i:nil="true"/>
      <ReturnDistinctRows>false</ReturnDistinctRows>
    </DataSetSettings>
    <DslQuerySettings>
      <AggregationFunction i:nil="true"/>
      <ComplexSourceQuestion i:nil="true"/>
      <Conditions i:nil="true"/>
      <Description i:nil="true"/>
      <Distinct>false</Distinct>
      <EnforceSecurity>true</EnforceSecurity>
      <ExternalParameters i:nil="true"/>
      <FormAnswerConditions i:nil="true"/>
      <Joined i:nil="true"/>
      <QueryLimit>500</QueryLimit>
      <ReferenceForm i:nil="true"/>
      <RequiredParameters i:nil="true"/>
      <RuntimeParameters i:nil="true"/>
      <SelectFields>
        <!-- SelectExpression elements here -->
      </SelectFields>
      <SortFields i:nil="true"/>
      <Source i:nil="true"/>
    </DslQuerySettings>
    <RunAsDSLQuery>true</RunAsDSLQuery>
  </QueryBuilder>
  <QueryIntegrationObjectType>None</QueryIntegrationObjectType>
  <QueryIntegrationType>KeyValue</QueryIntegrationType>
  <useDeletedFlagForParties>true</useDeletedFlagForParties>
</QueryIntegration>
```

---

## Worked Examples

### Example 1 — REST KeyValue dropdown (active offices)

```xml
<QueryIntegration>
  <Id>79</Id>
  <SchemaVersion>4300</SchemaVersion>
  <Active>true</Active>
  <DatasourceId>1</DatasourceId>
  <Draft>false</Draft>
  <IgnoreClientMatterSecurity>false</IgnoreClientMatterSecurity>
  <IsIntegrationTypeUserSelected>false</IsIntegrationTypeUserSelected>
  <Name>Get Active Offices</Name>
  <QueryBuilder>
    <DataSetSettings>
      <CollectionType>All</CollectionType>
      <DataSourceId>lookup</DataSourceId>
      <FieldNames>
        <string>[officeCode]</string>
        <string>[office]</string>
      </FieldNames>
      <OperationId>get-offices</OperationId>
      <ParameterExpressions>
        <RestParameterExpression>
          <Operator>Equals</Operator>
          <Parameter><IsExternal>false</IsExternal><IsRequired>false</IsRequired><Name>onlyActive</Name></Parameter>
          <Value>true</Value>
        </RestParameterExpression>
        <RestParameterExpression>
          <Operator>Equals</Operator>
          <Parameter><IsExternal>false</IsExternal><IsRequired>false</IsRequired><Name>sortBy</Name></Parameter>
          <Value>officeCode</Value>
        </RestParameterExpression>
      </ParameterExpressions>
      <ReturnDistinctRows>false</ReturnDistinctRows>
    </DataSetSettings>
    <RunAsDSLQuery>false</RunAsDSLQuery>
  </QueryBuilder>
  <QueryIntegrationObjectType>None</QueryIntegrationObjectType>
  <QueryIntegrationType>KeyValue</QueryIntegrationType>
  <useDeletedFlagForParties>false</useDeletedFlagForParties>
</QueryIntegration>
```

### Example 2 — DSL computed Value (sum two form fields)

```xml
<QueryIntegration>
  <Id>384</Id>
  <SchemaVersion>4300</SchemaVersion>
  <Active>true</Active>
  <DatasourceId>1</DatasourceId>
  <Draft>false</Draft>
  <IgnoreClientMatterSecurity>false</IgnoreClientMatterSecurity>
  <IsIntegrationTypeUserSelected>false</IsIntegrationTypeUserSelected>
  <Name>Budget Total</Name>
  <QueryBuilder>
    <DataSetSettings><ReturnDistinctRows>false</ReturnDistinctRows></DataSetSettings>
    <DslQuerySettings>
      <Distinct>false</Distinct>
      <EnforceSecurity>true</EnforceSecurity>
      <ExternalParameters>
        <EncodedExternalParameter><Value>#:#external#:#023fd1d4e22e480b0944c556f5c64385#:#Number#:#Fees#:#</Value></EncodedExternalParameter>
        <EncodedExternalParameter><Value>#:#external#:#81abbc25b95e4792b4946f97e4bd7ac3#:#Number#:#Costs#:#</Value></EncodedExternalParameter>
      </ExternalParameters>
      <QueryLimit>500</QueryLimit>
      <SelectFields>
        <SelectExpression>
          <ColumnName>Value</ColumnName>
          <Expression>Round([#:#external#:#023fd1d4e22e480b0944c556f5c64385#:#Number#:#Fees#:#] + [#:#external#:#81abbc25b95e4792b4946f97e4bd7ac3#:#Number#:#Costs#:#], 0)</Expression>
        </SelectExpression>
      </SelectFields>
    </DslQuerySettings>
    <RunAsDSLQuery>true</RunAsDSLQuery>
  </QueryBuilder>
  <QueryIntegrationObjectType>None</QueryIntegrationObjectType>
  <QueryIntegrationType>Value</QueryIntegrationType>
  <useDeletedFlagForParties>true</useDeletedFlagForParties>
</QueryIntegration>
```

### Example 3 — DSL KeyValue with join (parties with client ID)

```xml
<QueryIntegration>
  <Id>519</Id>
  <SchemaVersion>4300</SchemaVersion>
  <Active>true</Active>
  <DatasourceId>1</DatasourceId>
  <Draft>false</Draft>
  <IgnoreClientMatterSecurity>false</IgnoreClientMatterSecurity>
  <IsIntegrationTypeUserSelected>false</IsIntegrationTypeUserSelected>
  <Name>Get All Parties with client id by Search Term</Name>
  <QueryBuilder>
    <DataSetSettings><ReturnDistinctRows>false</ReturnDistinctRows></DataSetSettings>
    <DslQuerySettings>
      <Conditions>
        <CollectionType>All</CollectionType>
        <Conditions>
          <DataSetFilterCondition>
            <OrdinalIdx>1</OrdinalIdx>
            <Left><Value>#:#system#:#Parties#:#Parties#:#search-term-fts#:#</Value></Left>
            <Operator>Equals</Operator>
            <Right><DefaultValue i:nil="true"/><External>true</External><Name>p0</Name><ValueType>String</ValueType></Right>
          </DataSetFilterCondition>
        </Conditions>
      </Conditions>
      <Distinct>false</Distinct>
      <EnforceSecurity>true</EnforceSecurity>
      <Joined>
        <JoinedTable>
          <JoinType>Outer</JoinType>
          <Table><Alias>Clients</Alias><TableId>Clients</TableId></Table>
          <JoinCondition>
            <Left><Value>#:#system#:#Clients#:#Clients#:#PartyIdFk#:#</Value></Left>
            <Operator>Equals</Operator>
            <Right><Value>#:#system#:#Parties#:#Parties#:#Id#:#</Value></Right>
          </JoinCondition>
        </JoinedTable>
      </Joined>
      <QueryLimit>500</QueryLimit>
      <SelectFields>
        <SelectExpression>
          <ColumnName>Key</ColumnName>
          <Expression>[#:#system#:#Parties#:#Parties#:#Id#:#]</Expression>
        </SelectExpression>
        <SelectExpression>
          <ColumnName>Value</ColumnName>
          <Expression>case when [#:#system#:#Clients#:#Clients#:#ClientId#:#] is null
    then [#:#system#:#Parties#:#Parties#:#Name#:#] + ' [Party]'
    else [#:#system#:#Parties#:#Parties#:#Name#:#] + ' (' + [#:#system#:#Clients#:#Clients#:#ClientId#:#] + ') [Client]'
end</Expression>
        </SelectExpression>
      </SelectFields>
      <Source><Alias>Parties</Alias><TableId>Parties</TableId></Source>
    </DslQuerySettings>
    <RunAsDSLQuery>true</RunAsDSLQuery>
  </QueryBuilder>
  <QueryIntegrationObjectType>None</QueryIntegrationObjectType>
  <QueryIntegrationType>KeyValue</QueryIntegrationType>
  <useDeletedFlagForParties>true</useDeletedFlagForParties>
</QueryIntegration>
```

### Example 4 — DSL Datatable with sort

```xml
<QueryIntegration>
  <Id>322</Id>
  <SchemaVersion>4300</SchemaVersion>
  <Active>true</Active>
  <DatasourceId>1</DatasourceId>
  <Draft>false</Draft>
  <IgnoreClientMatterSecurity>false</IgnoreClientMatterSecurity>
  <IsIntegrationTypeUserSelected>false</IsIntegrationTypeUserSelected>
  <Name>Get Client Activity Details</Name>
  <QueryBuilder>
    <DataSetSettings><ReturnDistinctRows>false</ReturnDistinctRows></DataSetSettings>
    <DslQuerySettings>
      <Conditions>
        <CollectionType>All</CollectionType>
        <Conditions>
          <DataSetFilterCondition>
            <OrdinalIdx>1</OrdinalIdx>
            <Left><Value>#:#system#:#Clients#:#Clients#:#ClientId#:#</Value></Left>
            <Operator>In</Operator>
            <Right><DefaultValue i:nil="true"/><External>true</External><Name>p0</Name><ValueType>Dictionary</ValueType></Right>
          </DataSetFilterCondition>
        </Conditions>
      </Conditions>
      <Distinct>false</Distinct>
      <EnforceSecurity>true</EnforceSecurity>
      <Joined>
        <JoinedTable>
          <Table><Alias>Activities</Alias><TableId>Activities</TableId></Table>
          <JoinCondition>
            <Left><Value>#:#system#:#Activities#:#Activities#:#ComponentId#:#</Value></Left>
            <Operator>Equals</Operator>
            <Right><Value>#:#system#:#Clients#:#Clients#:#ComponentIdFk#:#</Value></Right>
          </JoinCondition>
        </JoinedTable>
      </Joined>
      <QueryLimit>500</QueryLimit>
      <SelectFields>
        <SelectExpression><ColumnName>Client Id</ColumnName><Expression>[#:#system#:#Clients#:#Clients#:#ClientId#:#]</Expression></SelectExpression>
        <SelectExpression><ColumnName>Client Name</ColumnName><Expression>[#:#system#:#Clients#:#Clients#:#Name#:#]</Expression></SelectExpression>
        <SelectExpression><ColumnName>Activity Type</ColumnName><Expression>[#:#system#:#Activities#:#Activities#:#ActivityType#:#]</Expression></SelectExpression>
        <SelectExpression><ColumnName>Activity Description</ColumnName><Expression>[#:#system#:#Activities#:#Activities#:#Description#:#]</Expression></SelectExpression>
        <SelectExpression><ColumnName>Created On</ColumnName><Expression>[#:#system#:#Activities#:#Activities#:#CreatedOn#:#]</Expression></SelectExpression>
      </SelectFields>
      <SortFields>
        <SortField>
          <Direction>ASCENDING</Direction>
          <Field><Value>#:#system#:#Clients#:#Clients#:#ClientId#:#</Value></Field>
        </SortField>
      </SortFields>
      <Source><Alias>Clients</Alias><TableId>Clients</TableId></Source>
    </DslQuerySettings>
    <RunAsDSLQuery>true</RunAsDSLQuery>
  </QueryBuilder>
  <QueryIntegrationObjectType>None</QueryIntegrationObjectType>
  <QueryIntegrationType>Datatable</QueryIntegrationType>
  <useDeletedFlagForParties>true</useDeletedFlagForParties>
</QueryIntegration>
```

---

## Workflow

Work through the following steps with the user. Use a todo list to track progress.

### 1. Gather Requirements

Ask the user:
- **Name**: What should this dataset be called?
- **Purpose**: What will it power? (dropdown, calculated value, data grid)
- **Result type**: KeyValue / Value / Datatable
- **Data source**: System tables, virtual tables, or a REST API endpoint?
- **Filters**: What conditions filter the data? Any runtime/user-supplied parameters?
- **Fields**: What columns should be returned?
- **Joins**: Does the query need data from multiple tables?
- **Sort**: Should results be sorted?

### 2. Choose Query Mode

- Use **REST** when the data comes from a standard Wilco API endpoint (matters, clients, offices, users, etc.)
- Use **DSL** when you need custom logic, joins, computed expressions, or data from system/virtual tables

### 3. Build the XML

Using the templates and examples above, construct the full `<QueryIntegration>` XML. Key checklist:
- [ ] Assign a unique `Id` (check existing IDs in the file to pick the next available)
- [ ] Set `QueryIntegrationType` to match the intended use
- [ ] For DSL: set `useDeletedFlagForParties` to `true`
- [ ] For DSL KeyValue: columns must be named exactly `Key` and `Value`
- [ ] For DSL Value: column must be named exactly `Value`
- [ ] Wrap all field references in `[...]`
- [ ] Set `i:nil="true"` on unused optional elements (or omit them)
- [ ] Set `EnforceSecurity` appropriately (default `true`)

### 4. Validate

Review the generated XML with the user. Check:
- Field reference tokens are correctly formatted
- Join conditions reference fields from the correct tables
- External parameter GUIDs match the form fields being referenced
- `QueryLimit` is appropriate for the expected data volume

### 5. Add to the .ixt File

The new `<QueryIntegration>` element goes inside the `<QueryIntegrations>` collection in `definitions.xml`. After updating, re-package:

```bash
# Update definitions.xml inside the .ixt (zip) file
zip "ExportDataSets.ixt" definitions.xml
```

Then import the updated `.ixt` into Wilco via the Template Import UI.

### 6. Commit

Commit the updated `.ixt` file to the repo branch with a clear message describing the new dataset.
