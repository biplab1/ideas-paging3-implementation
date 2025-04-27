# ideas-paging3-implementation
List of ideas to incorporate/ remove paging 3 from Mifos android-client app (mifos-x-field-officer-app)

## Introduction
Paging 3 supports only three platforms as of now- android, iOS, desktop (JVM). 

## Usage in Mifos Android-Client
Paging 3 is used in 2 layers - data, UI (viewModel, screen). One useCase uses paging library in the domain layer `GroupListPagingDataSource.kt`.

## Our Requirements
Our project requires building for 5 platforms- android, iOS, desktop, JS and wasmJS. 
Paging 3 doesn't support web (JS and wasmJS)

## Idea 1
Do not build targets for the web currently, Paging 3 components will work as it is out of the box. 
### Pros
Data layer becomes ready to merge without much work

### Cons
If we incorporate the web targets in the future, one needs to resolve the paging 3 error since it does not support the web for now. 

## Idea 2
Put all the paging 3 library using classes in the androidMain- pagingSource, repository, repositoryImpl. We focus on android app only so that we can get the features migrated to CMP. 

This should also work if we put the classes, as it is, in nativeMain (iOS) and desktopMain, but lot of code repetition. One workaround would be to create a sharedNativeMain, which has all those classes and androidMain, nativeMain and desktopMain simply use it from there avoiding code repetition.

### Steps for implementation
<ol>
  <li>Move classes that related to or using paging library to androidMain</li>
  <li>Move part of koin module initiation of the repository implmentation (`RepositoryModule.kt`) using paging 3 library to androidMain (not yet tried), since DI in commonMain cannot see the repository and its implementation in </li>
  <li></li>
</ol>

### Pros

Again it works out of box and 

### Cons


## Idea 3
Refactor code to support Kotlin Multiplatform for all platforms. 

### Steps for implementation

## Code (Original)
We use the `ClientListRepository.kt` and `ClientListRepositoryImp.kt` as an example:
```kotlin
// ClientListRepository.kt (:core:data:repository)
package com.mifos.core.data.repository

import androidx.paging.PagingData
import com.mifos.core.common.utils.Page
import com.mifos.room.entities.client.ClientEntity
import kotlinx.coroutines.flow.Flow

interface ClientListRepository {

    fun getAllClients(): Flow<PagingData<ClientEntity>>

    fun allDatabaseClients(): Flow<Page<ClientEntity>>
}
```
```kotlin
// ClientListRepositoryImp.kt (:core:data:repositoryImp)
package com.mifos.core.data.repositoryImp

import androidx.paging.Pager
import androidx.paging.PagingConfig
import androidx.paging.PagingData
import com.mifos.core.common.utils.Page
import com.mifos.core.data.pagingSource.ClientListPagingSource
import com.mifos.core.data.repository.ClientListRepository
import com.mifos.core.network.datamanager.DataManagerClient
import com.mifos.room.entities.client.ClientEntity
import kotlinx.coroutines.flow.Flow

class ClientListRepositoryImp(
    private val dataManagerClient: DataManagerClient,
) : ClientListRepository {

    override fun getAllClients(): Flow<PagingData<ClientEntity>> {
        return Pager(
            config = PagingConfig(
                pageSize = 10,
            ),
            pagingSourceFactory = {
                ClientListPagingSource(dataManagerClient)
            },
        ).flow
    }

    override fun allDatabaseClients(): Flow<Page<ClientEntity>> {
        return dataManagerClient.allDatabaseClients
    }
}
```
```kotlin
// ClientListPagingSource.kt (:core:data:pagingSource)
package com.mifos.core.data.pagingSource

import androidx.paging.PagingSource
import androidx.paging.PagingState
import com.mifos.core.network.datamanager.DataManagerClient
import com.mifos.room.entities.client.ClientEntity
import kotlinx.coroutines.flow.firstOrNull
import kotlinx.coroutines.flow.map

class ClientListPagingSource(
    private val dataManagerClient: DataManagerClient,
) : PagingSource<Int, ClientEntity>() {

    override fun getRefreshKey(state: PagingState<Int, ClientEntity>): Int? {
        return state.anchorPosition?.let { position ->
            state.closestPageToPosition(position)?.prevKey?.plus(10) ?: state.closestPageToPosition(
                position,
            )?.nextKey?.minus(10)
        }
    }

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, ClientEntity> {
        val position = params.key ?: 0
        return try {
            val getClients = getClientList(position)
            val clientList = getClients.first
            val totalClients = getClients.second
            val clientDbList = getClientDbList()
            val clientListWithSync = getClientListWithSync(clientList, clientDbList)
            LoadResult.Page(
                data = clientListWithSync,
                prevKey = if (position <= 0) null else position - 10,
                nextKey = if (position >= totalClients) null else position + 10,
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }

    private suspend fun getClientList(position: Int): Pair<List<ClientEntity>, Int> {
        val response = dataManagerClient.getAllClients(position, 10)
        return Pair(response.pageItems, response.totalFilteredRecords)
    }

    private suspend fun getClientDbList(): List<ClientEntity> {
        return dataManagerClient.allDatabaseClients
            .map { it.pageItems }
            .firstOrNull() ?: emptyList()
    }

    private fun getClientListWithSync(
        clientList: List<ClientEntity>,
        clientDbList: List<ClientEntity>,
    ): List<ClientEntity> {
        if (clientDbList.isNotEmpty()) {
            clientList.forEach { client ->
                clientDbList.forEach { clientDb ->
                    if (client.id == clientDb.id) {
                        // TODO:: Unused result of data class copy, fix this implementation
                        client.copy(sync = true)
                    }
                }
            }
        }
        return clientList
    }
}
```
### Refactoring to Support KMP
A new data class `PageResult.kt` added
```kotlin
// PageResult.kt (:core:model:objects)
package com.mifos.core.model.objects

import com.mifos.core.model.utils.Parcelable
import com.mifos.core.model.utils.Parcelize
import kotlinx.serialization.Serializable

@Parcelize
@Serializable
data class PageResult<T>(
    val data: List<T>,
    val prevKey: Int?,
    val nextKey: Int?,
) : Parcelable

```
```kotlin
// ClientListPaginator.kt (:core:data:pagingSource)

package com.mifos.core.data.pagingSource

import com.mifos.core.model.objects.PageResult
import com.mifos.core.network.datamanager.DataManagerClient
import com.mifos.room.entities.client.ClientEntity
import kotlinx.coroutines.flow.firstOrNull
import kotlinx.coroutines.flow.map

class ClientListPaginator(
    private val dataManagerClient: DataManagerClient,
    private val pageSize: Int,
) {
    suspend fun loadPage(position: Int): PageResult<ClientEntity> {
        val getClients = getClientList(position)
        val clientList = getClients.first
        val totalClients = getClients.second
        val clientDbList = getClientDbList()
        val clientListWithSync = getClientListWithSync(clientList, clientDbList)

        return PageResult(
            data = clientListWithSync,
            prevKey = if (position <= 0) null else position - pageSize,
            nextKey = if (position + pageSize >= totalClients) null else position + pageSize,
        )
    }

    private suspend fun getClientList(position: Int): Pair<List<ClientEntity>, Int> {
        val response = dataManagerClient.getAllClients(position, pageSize)
        return response.pageItems to response.totalFilteredRecords
    }

    private suspend fun getClientDbList(): List<ClientEntity> {
        return dataManagerClient.allDatabaseClients
            .map { it.pageItems }
            .firstOrNull() ?: emptyList()
    }

    private fun getClientListWithSync(
        clientList: List<ClientEntity>,
        clientDbList: List<ClientEntity>,
    ): List<ClientEntity> {
        val dbIds = clientDbList.map { it.id }.toSet()
        return clientList.map { client ->
            if (client.id in dbIds) client.copy(sync = true) else client
        }
    }
}
```
Files added to androidMain:
```kotlin
// AndroidClientListPagingSource.kt (:core:data:pagingSource)

package com.mifos.core.data.pagingSource

import androidx.paging.PagingSource
import androidx.paging.PagingState
import com.mifos.room.entities.client.ClientEntity

class ClientListPagingSource(
    private val paginator: ClientListPaginator,
) : PagingSource<Int, ClientEntity>() {

    override fun getRefreshKey(state: PagingState<Int, ClientEntity>): Int? {
        return state.anchorPosition?.let { position ->
            state.closestPageToPosition(position)?.prevKey?.plus(10)
                ?: state.closestPageToPosition(position)?.nextKey?.minus(10)
        }
    }

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, ClientEntity> {
        return try {
            val result = paginator.loadPage(params.key ?: 0)
            LoadResult.Page(
                data = result.data,
                prevKey = result.prevKey,
                nextKey = result.nextKey,
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }
}
```
```kotlin
// AndroidClientListRepository.kt

package com.mifos.core.data.repository

import androidx.paging.PagingData
import com.mifos.room.entities.client.ClientEntity
import kotlinx.coroutines.flow.Flow

interface AndroidClientListRepository : ClientListRepository {
   override fun getAllClients(): Flow<PagingData<ClientEntity>>
}
```
```kotlin
// AndroidClientListRepositoryImp.kt
package com.mifos.core.data.repositoryImp

import androidx.paging.Pager
import androidx.paging.PagingConfig
import androidx.paging.PagingData
import com.mifos.core.common.utils.DataState
import com.mifos.core.common.utils.Page
import com.mifos.core.common.utils.asDataStateFlow
import com.mifos.core.data.pagingSource.ClientListPaginator
import com.mifos.core.data.repository.AndroidClientListRepository
import com.mifos.core.network.datamanager.DataManagerClient
import com.mifos.room.entities.client.ClientEntity
import kotlinx.coroutines.CoroutineDispatcher
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flowOn

class AndroidClientListRepositoryImp(
   private val dataManagerClient: DataManagerClient,
   private val ioDispatcher: CoroutineDispatcher,
) : AndroidClientListRepository {

   override fun getAllClients(): Flow<PagingData<ClientEntity>> {
       return Pager(
           config = PagingConfig(pageSize = 10),
           pagingSourceFactory = { ClientListPaginator(dataManagerClient, pageSize = 10) },
       ).flow
   }

   override fun allDatabaseClients(): Flow<DataState<Page<ClientEntity>>> {
       return dataManagerClient.allDatabaseClients
           .asDataStateFlow().flowOn(ioDispatcher)
   }
}
```

### Roadblocks
<ol>
  <li>How to handle the repository and repository implemetations using paging 3 in KMP if we use expect/actual. We need to add the return datatype in the expect function/ class/ interface, however, that return type belongs to paging 3</li>
<li>We can use a mapper to handle conversion between our custome PagingResult class and PagingData class (paging 3)</li>
  <li>How will the UI layer viewModels access the PagingData (part of paging library), since we cannot put that viewModel in commonMain</li>
</ol>

## Idea 3
Mr. Rajan (our team lead) suggested to bypass the paging library usage and delegate the work of incorporating the paging library in a separate ticket.

### Pros
Easier to implement compared to the other ideas.

### Cons
UI layer uses paging library to fetch data, that would require a lot of refactoring code in the UI layer, both screen as well as ViewModel.
