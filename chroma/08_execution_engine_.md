# Chapter 8: Execution Engine

In the previous chapter, [Chapter 7: Segments](07_segments_.md), we learned that your collection's data – the vectors, documents, and metadata – isn't stored in one big chunk, but is organized into specialized storage units called **Segments**. These segments are designed for efficient storage and retrieval of their specific data types.

But how does Chroma actually answer your questions or find similar items, especially when your queries are complex? For example, what happens when you ask Chroma to "find documents similar to X, but *only* among those that have metadata 'status' equal to 'processed' and document content containing 'report'"?

Answering a query like this requires coordinating work: first finding items that match the metadata and document filters (handled by [Metadata Segments](07_segments_.md)), and then performing a similarity search *only* among those filtered items (handled by [Vector Segments](07_segments_.md)).

This coordination, planning, and execution of complex queries is the job of the **Execution Engine**.

Think of the Execution Engine as the "query processing center" or the "factory foreman" for your data requests. When a query comes in (like your complex search request), the Execution Engine doesn't just blindly search everywhere. Instead, it:

1.  Analyzes the request.
2.  Creates a detailed **plan** (a blueprint) for how to fulfill the request efficiently.
3.  Breaks the plan down into specific **operators** (individual steps like filtering or searching).
4.  Uses **executors** (specialized workers) to carry out these steps by interacting with the correct **Segments** to fetch and process the data.
5.  Collects the results from the different steps and assembles the final answer to your query.

It's like the factory needs to build a custom product: The planning department creates the blueprint (Plan), the blueprint specifies different tasks (Operators), and specialized workers (Executors) go to the right workstations/storage units ([Segments]) to get parts and perform their task according to the blueprint.

## Key Concepts: Planning and Operators

The Execution Engine works by transforming a user query into a structured **Plan**, composed of various **Operators**.

1.  **Query Request:** This is what you send via the [Client API](01_client_api_.md), like `collection.query(query_texts=..., where=..., where_document=..., n_results=...)`.

2.  **Execution Plan:** The Execution Engine component takes this request and generates a `Plan` object. A plan is a structured representation of the query, outlining the overall steps needed. For example, a search query with filtering might result in a plan that says "Scan the collection, then apply the metadata filter, then apply the document filter, then perform a KNN search on the remaining items, and finally, return the top N results including documents and metadata." Chroma defines different types of plans for different operations (like `CountPlan`, `GetPlan`, `KNNPlan`).

    *   Look at the `KNNPlan` definition in `chromadb/execution/expression/plan.py` (simplified):
        ```python
        # Simplified snippet from chromadb/execution/expression/plan.py
        from dataclasses import dataclass, field
        from chromadb.execution.expression.operator import KNN, Filter, Projection, Scan

        @dataclass
        class KNNPlan:
            scan: Scan             # Describes the collection/segments to operate on
            knn: KNN               # Details about the KNN search (query embeddings, n_results)
            filter: Filter = field(default_factory=Filter) # Details about filters (where, where_document, ids)
            projection: Projection = field(default_factory=Projection) # Which fields to return (documents, metadata, etc.)

        # CountPlan and GetPlan are similar dataclasses for those operations
        @dataclass
        class CountPlan:
            scan: Scan

        @dataclass
        class GetPlan:
            scan: Scan
            filter: Filter = field(default_factory=Filter)
            limit: Limit = field(default_factory=Limit)
            projection: Projection = field(default_factory=Projection)
        ```
        These dataclasses are simple data structures that hold all the information needed to describe a query operation.

3.  **Operators:** The Plan is composed of `Operator` objects. Each Operator represents a distinct logical step or piece of information required by the plan.

    *   Look at the definitions of some common Operators in `chromadb/execution/expression/operator.py` (simplified):
        ```python
        # Simplified snippet from chromadb/execution/expression/operator.py
        from dataclasses import dataclass
        from typing import Optional
        from chromadb.api.types import Embeddings, IDs, Include
        from chromadb.types import Collection, RequestVersionContext, Segment, Where, WhereDocument

        @dataclass
        class Scan:
            collection: Collection # Info about the collection
            knn: Segment         # The Vector Segment for this collection
            metadata: Segment    # The Metadata Segment for this collection
            record: Segment      # The Record Segment for this collection
            # ... other segments ...

            @property
            def version(self) -> RequestVersionContext:
                # Provides version info from the collection
                pass

        @dataclass
        class Filter:
            user_ids: Optional[IDs] = None       # Filter by specific item IDs
            where: Optional[Where] = None        # Filter by metadata conditions
            where_document: Optional[WhereDocument] = None # Filter by document content conditions

        @dataclass
        class KNN:
            embeddings: Embeddings # The query embedding vectors
            fetch: int             # Number of nearest neighbors to fetch (k)

        @dataclass
        class Limit:
            skip: int = 0         # Number of results to skip (for pagination)
            fetch: Optional[int] = None # Maximum number of results to return (for pagination)

        @dataclass
        class Projection:
            document: bool = False # Should include documents in results?
            embedding: bool = False # Should include embeddings in results?
            metadata: bool = False # Should include metadata in results?
            rank: bool = False      # Should include distances/similarity score?
            uri: bool = False       # Should include URIs?

            @property
            def included(self) -> Include:
                 # Helper to format includes for the QueryResult
                 pass

        ```
        These dataclasses represent the specific components or criteria within a query plan. The `Scan` operator tells the Executor *which* segments it needs to work with for this collection. The `Filter` operator describes any conditions on metadata or documents. The `KNN` operator describes the vector part of the search. `Limit` and `Projection` specify how the results should be shaped.

## You Don't Directly Interact with the Execution Engine

Like the [Server API](03_server_api_.md), [System Database (SysDB)](05_system_database__sysdb__.md), and [Segments](07_segments_.md), the Execution Engine is an internal Chroma component. You interact with it indirectly through the [Client API](01_client_api_.md).

When you call `collection.query()`, `collection.get()`, or `collection.count()`, the [Client API](01_client_api_.md) sends the request to the [Server API](03_server_api_.md). The [Server API](03_server_api_.md) then uses the Execution Engine component to process that request.

Let's revisit the `collection.query` example from [Chapter 1: Client API](01_client_api_.md), slightly expanding the internal view from [Chapter 3: Server API](03_server_api_.md).

```python
# Example from Chapter 1
results = collection.query(
    query_texts=["Tell me about vector databases"], # Query text
    n_results=2,                                # Number of results
    # Imagine adding filters here:
    # where={"source": "manual_entry"},
    # where_document={"$contains": "database"},
)
```

When you execute this, the following happens internally:

1.  The [Client API](01_client_api_.md) validates the input, gets the collection's configured [Embedding Function](02_embedding_function_.md) (if you provided `query_texts`), and embeds the query text(s) into `query_embeddings`.
2.  The [Client API](01_client_api_.md) sends the request to the [Server API](03_server_api_.md) with the `query_embeddings`, `n_results`, filters (`where`, `where_document`), and other parameters.
3.  The [Server API](03_server_api_.md) component receives the request.
4.  The [Server API](03_server_API_.md) interacts with the [System Database (SysDB)](05_system_database__sysdb__.md) (perhaps via the Segment Manager) to get information about the collection and the specific [Segments](07_segments_.md) associated with it (Vector Segment, Metadata Segment).
5.  The [Server API](03_server_API_.md) creates a **Plan** object (specifically, a `KNNPlan` in this case), populating it with the query embeddings, `n_results`, filter details, and the relevant segment information.
6.  The [Server API](03_server_API_.md) passes this `KNNPlan` to the **Execution Engine** component (specifically, to its `knn` method).
7.  The Execution Engine component receives the `KNNPlan`. It then executes the operators defined in the plan by calling the appropriate methods on the [Segment](07_segments_.md) instances it needs (obtained via the Segment Manager).
8.  The Execution Engine combines the results from the segments (e.g., getting IDs and distances from the Vector Segment, then fetching documents and metadata for those IDs from the Metadata Segment, applying any additional filters), formats them, and returns them to the [Server API](03_server_api_.md).
9.  The [Server API](03_server_api_.md) returns the results to the [Client API](01_client_api_.md).
10. The [Client API](01_client_api_.md) structures the results into the `QueryResult` dictionary you receive.

## Under the Hood: How the Execution Engine Uses Segments

The core job of the Execution Engine is to take a `Plan` (which contains `Operator`s and references to [Segments]) and execute it.

The Execution Engine component is an implementation of the `Executor` abstract class, defined in `chromadb/execution/executor/abstract.py`. It has methods corresponding to the main query types:

```python
# Simplified snippet from chromadb/execution/executor/abstract.py
from abc import abstractmethod
from chromadb.api.types import GetResult, QueryResult
from chromadb.config import Component
from chromadb.execution.expression.plan import CountPlan, GetPlan, KNNPlan

class Executor(Component): # The Executor is a Component managed by the System
    @abstractmethod
    def count(self, plan: CountPlan) -> int:
        """Execute a count plan."""
        pass

    @abstractmethod
    def get(self, plan: GetPlan) -> GetResult:
        """Execute a get plan."""
        pass

    @abstractmethod
    def knn(self, plan: KNNPlan) -> QueryResult:
        """Execute a K-Nearest Neighbors search plan."""
        pass
```
Chroma provides different implementations of the `Executor` interface depending on the deployment mode:

*   **`LocalExecutor`:** Used by `EphemeralClient` and `PersistentClient`. This executor runs within the same process and directly interacts with local [Segment](07_segments_.md) instances (like `PersistentLocalHnswSegment` and `SqliteMetadataSegment`) provided by the `LocalSegmentManager`.
*   **`DistributedExecutor`:** Used in a distributed Chroma cluster. This executor does *not* run the query logic itself. Instead, it acts as a client, sending the query plan over the network (using gRPC) to remote services that host the segments and run their own executors.

Let's look at a simplified version of the `LocalExecutor`'s `knn` method (`chromadb/execution/executor/local.py`) to see how it uses the Segment Manager and Segments:

```python
# Simplified snippet from chromadb/execution/executor/local.py
from typing import Optional, Sequence
from overrides import overrides
from chromadb.api.types import GetResult, Metadata, QueryResult
from chromadb.config import System
from chromadb.execution.executor.abstract import Executor
from chromadb.execution.expression.plan import CountPlan, GetPlan, KNNPlan
from chromadb.segment import MetadataReader, VectorReader # Import segment interfaces
from chromadb.segment.impl.manager.local import LocalSegmentManager # Import Segment Manager
from chromadb.types import Collection, VectorQuery, VectorQueryResult

class LocalExecutor(Executor):
    _manager: LocalSegmentManager # LocalExecutor needs the Segment Manager

    def __init__(self, system: System):
        super().__init__(system)
        # Get the Segment Manager component from the System
        self._manager = self.require(LocalSegmentManager)

    @overrides # Indicates this implements the method from the base class
    def knn(self, plan: KNNPlan) -> QueryResult:
        # Plan details are available via plan.filter, plan.knn, plan.projection, etc.

        # 1. Handle filtering if any is specified in the plan
        prefiltered_ids = None
        if plan.filter.user_ids or plan.filter.where or plan.filter.where_document:
            # Get the Metadata Segment for this collection using the manager
            metadata_segment = self._metadata_segment(plan.scan.collection) # Calls self._manager.get_segment(...)

            # Ask the Metadata Segment to find IDs matching the filter criteria
            records = metadata_segment.get_metadata(
                request_version_context=plan.scan.version,
                where=plan.filter.where,
                where_document=plan.filter.where_document,
                ids=plan.filter.user_ids,
                include_metadata=False, # Only need IDs here
            )
            prefiltered_ids = [r["id"] for r in records] # Collect the matching IDs

        # 2. Perform KNN search using the Vector Segment
        knns: Sequence[Sequence[VectorQueryResult]] = [[]] * len(plan.knn.embeddings)
        # Only query vectors if there were no filters or if filters returned some IDs
        if prefiltered_ids is None or len(prefiltered_ids) > 0:
            # Get the Vector Segment for this collection using the manager
            vector_segment = self._vector_segment(plan.scan.collection) # Calls self._manager.get_segment(...)

            # Prepare the VectorQuery object based on the KNN operator from the plan
            query = VectorQuery(
                vectors=plan.knn.embeddings,
                k=plan.knn.fetch,
                allowed_ids=prefiltered_ids, # Pass pre-filtered IDs to the vector segment!
                include_embeddings=plan.projection.embedding, # Include embeddings if requested in projection
                request_version_context=plan.scan.version,
            )
            # Ask the Vector Segment to perform the KNN search
            knns = vector_segment.query_vectors(query) # This calls the segment's HNSW/indexing logic

        # 3. Process results based on Projection (documents, metadata, distances)
        ids = [[r["id"] for r in result] for result in knns]
        embeddings = None
        documents = None
        metadatas = None
        distances = None

        if plan.projection.rank: # If distances requested
            distances = [[r["distance"] for r in result] for result in knns]

        # If documents or metadata requested, need to fetch them from the Metadata Segment
        if plan.projection.document or plan.projection.metadata or plan.projection.uri:
            # Get all unique IDs from the KNN results
            merged_ids = list(set([id for result in ids for id in result]))

            if len(merged_ids) > 0:
                 # Get the Metadata Segment again (or use the existing reference)
                 metadata_segment = self._metadata_segment(plan.scan.collection)
                 # Ask the Metadata Segment for the documents and metadata for these IDs
                 hydrated_records = metadata_segment.get_metadata(
                    request_version_context=plan.scan.version,
                    ids=merged_ids, # Fetching specific IDs from vector results
                    include_metadata=True,
                 )
                 # Map IDs to their documents/metadata for easy lookup
                 metadata_by_id = {r["id"]: r["metadata"] for r in hydrated_records}

                 if plan.projection.document:
                     documents = [
                        [_doc(metadata_by_id.get(id, None)) for id in result]
                        for result in ids
                     ]
                 if plan.projection.metadata:
                     metadatas = [
                        [_clean_metadata(metadata_by_id.get(id, None)) for id in result]
                        for result in ids
                     ]
                 # ... similar logic for URI ...

        # 4. Format and return the final QueryResult
        return QueryResult(
            ids=ids,
            embeddings=embeddings,
            documents=documents,
            metadatas=metadatas,
            distances=distances,
            # Included field indicates which fields were requested
            included=plan.projection.included,
            uris=None, # Simplified
            data=None, # Simplified
        )

    # Helper methods to get segment instances from the manager
    def _metadata_segment(self, collection: Collection) -> MetadataReader:
        return self._manager.get_segment(collection.id, MetadataReader)

    def _vector_segment(self, collection: Collection) -> VectorReader:
        return self._manager.get_segment(collection.id, VectorReader)

    # count and get methods would follow a similar pattern, calling the relevant Segment methods
    # @overrides
    # def get(self, plan: GetPlan) -> GetResult:
    #     # Uses self._metadata_segment().get_metadata(...) primarily
    #     pass
    # @overrides
    # def count(self, plan: CountPlan) -> int:
    #     # Uses self._metadata_segment().count(...)
    #     pass

```
This simplified `LocalExecutor.knn` method demonstrates the core responsibility of the Execution Engine:

1.  Receive a `Plan` object.
2.  Use the `Plan`'s `Filter` operator to potentially pre-filter IDs by asking the [Metadata Segment](07_segments_.md) for matching records.
3.  Use the `Plan`'s `KNN` operator to perform the core similarity search by asking the [Vector Segment](07_segments_.md) for nearest neighbors, optionally restricting the search to the pre-filtered IDs.
4.  Use the `Plan`'s `Projection` operator to determine what additional information (documents, metadata) is needed for the results and fetch that information from the [Metadata Segment](07_segments_.md) based on the IDs returned by the vector search.
5.  Combine all the pieces of information and format the final `QueryResult`.

In essence, the Execution Engine is the traffic controller and assembler for your queries, directing requests to the right [Segments](07_segments_.md), ensuring filters are applied correctly, searches are performed, and the results are put together before being returned to the user.

## Distributed Execution

The `DistributedExecutor` (`chromadb/execution/executor/distributed.py`) implements the *same* `Executor` interface (`count`, `get`, `knn`). However, instead of calling methods on local segment instances, its methods build gRPC requests using protobufs (like `convert.to_proto_knn_plan(plan)`) and send them to remote services using gRPC stubs (`self._get_stub(endpoint)`). These remote services are responsible for running the equivalent execution logic on the segments they host.

```python
# Simplified snippet from chromadb/execution/executor/distributed.py
import grpc
from overrides import overrides
from chromadb.execution.executor.abstract import Executor
from chromadb.proto import convert # Used to convert Plans to protobuf messages
from chromadb.proto.query_executor_pb2_grpc import QueryExecutorStub # gRPC client stub

class DistributedExecutor(Executor):
    _grpc_stub_pool: Dict[str, QueryExecutorStub] # Stubs to connect to remote executors

    def __init__(self, system: System):
        super().__init__(system)
        # ... initialize grpc stub pool, get remote endpoints from SegmentManager ...

    @overrides
    def knn(self, plan: KNNPlan) -> QueryResult:
        # 1. Get appropriate remote endpoints for the query (potentially multiple for replication)
        endpoints = self._get_grpc_endpoints(plan.scan)

        # 2. Prepare the protobuf request message from the Plan
        proto_plan = convert.to_proto_knn_plan(plan)

        # 3. Send the request to a remote endpoint using the gRPC stub
        #    (Includes retry logic for resilience)
        response = self._round_robin_retry(
            [self._get_stub(endpoint).KNN for endpoint in endpoints], proto_plan
        ) # Calls remote KNN service

        # 4. Convert the protobuf response back to Chroma's Python types
        results = convert.from_proto_knn_batch_result(response)

        # 5. Format and return the final QueryResult (similar to LocalExecutor)
        ids = [[record["record"]["id"] for record in records] for records in results]
        # ... extract embeddings, documents, metadatas, distances from results ...

        return QueryResult(
            ids=ids,
            # ... populate other fields ...
        )

    # _get_grpc_endpoints, _get_stub, _round_robin_retry methods omitted
    # count and get methods follow similar pattern, calling remote Count/Get services
```
This shows that the Executor interface allows Chroma to swap out the execution *mechanism* (local function calls vs. remote gRPC calls) while the Server API and the Plan structure remain consistent.

## Conclusion

In this chapter, we explored the **Execution Engine**, the crucial internal component responsible for processing user queries in Chroma. We learned that it transforms a query request into a structured **Plan**, which is composed of various **Operators** representing specific steps like scanning, filtering, and KNN search.

We saw that the Execution Engine uses **executors** (like `LocalExecutor` or `DistributedExecutor`) to run these plans by coordinating with the appropriate **[Segments](07_segments_.md)** to retrieve and process data. The Execution Engine ensures complex queries involving filtering and similarity search are handled efficiently by correctly orchestrating the operations across different segment types.

While you don't directly interact with the Execution Engine, understanding its role clarifies how your `collection.query`, `get`, and `count` calls are broken down, planned, and executed within Chroma to deliver your results.

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)