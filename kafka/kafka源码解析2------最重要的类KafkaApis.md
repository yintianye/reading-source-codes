# kafka源码解析2------最重要的类KafkaApis

这个类可以说是最重要的了。它里面有对于所有api的定义和处理方法。

* api定义

  KafkaApis中的handle方法是处理RequestChannel.Request的入口，先看看源码：

  ```scala
    def handle(request: RequestChannel.Request) {
      try {
        trace(s"Handling request:${request.requestDesc(true)} from connection ${request.context.connectionId};" +
          s"securityProtocol:${request.context.securityProtocol},principal:${request.context.principal}")
        request.header.apiKey match {
          case ApiKeys.PRODUCE => handleProduceRequest(request)
          case ApiKeys.FETCH => handleFetchRequest(request)
          case ApiKeys.LIST_OFFSETS => handleListOffsetRequest(request)
          case ApiKeys.METADATA => handleTopicMetadataRequest(request)
          case ApiKeys.LEADER_AND_ISR => handleLeaderAndIsrRequest(request)
          case ApiKeys.STOP_REPLICA => handleStopReplicaRequest(request)
          case ApiKeys.UPDATE_METADATA => handleUpdateMetadataRequest(request)
          case ApiKeys.CONTROLLED_SHUTDOWN => handleControlledShutdownRequest(request)
          case ApiKeys.OFFSET_COMMIT => handleOffsetCommitRequest(request)
          case ApiKeys.OFFSET_FETCH => handleOffsetFetchRequest(request)
          case ApiKeys.FIND_COORDINATOR => handleFindCoordinatorRequest(request)
          case ApiKeys.JOIN_GROUP => handleJoinGroupRequest(request)
          case ApiKeys.HEARTBEAT => handleHeartbeatRequest(request)
          case ApiKeys.LEAVE_GROUP => handleLeaveGroupRequest(request)
          case ApiKeys.SYNC_GROUP => handleSyncGroupRequest(request)
          case ApiKeys.DESCRIBE_GROUPS => handleDescribeGroupRequest(request)
          case ApiKeys.LIST_GROUPS => handleListGroupsRequest(request)
          case ApiKeys.SASL_HANDSHAKE => handleSaslHandshakeRequest(request)
          case ApiKeys.API_VERSIONS => handleApiVersionsRequest(request)
          case ApiKeys.CREATE_TOPICS => handleCreateTopicsRequest(request)
          case ApiKeys.DELETE_TOPICS => handleDeleteTopicsRequest(request)
          case ApiKeys.DELETE_RECORDS => handleDeleteRecordsRequest(request)
          case ApiKeys.INIT_PRODUCER_ID => handleInitProducerIdRequest(request)
          case ApiKeys.OFFSET_FOR_LEADER_EPOCH => handleOffsetForLeaderEpochRequest(request)
          case ApiKeys.ADD_PARTITIONS_TO_TXN => handleAddPartitionToTxnRequest(request)
          case ApiKeys.ADD_OFFSETS_TO_TXN => handleAddOffsetsToTxnRequest(request)
          case ApiKeys.END_TXN => handleEndTxnRequest(request)
          case ApiKeys.WRITE_TXN_MARKERS => handleWriteTxnMarkersRequest(request)
          case ApiKeys.TXN_OFFSET_COMMIT => handleTxnOffsetCommitRequest(request)
          case ApiKeys.DESCRIBE_ACLS => handleDescribeAcls(request)
          case ApiKeys.CREATE_ACLS => handleCreateAcls(request)
          case ApiKeys.DELETE_ACLS => handleDeleteAcls(request)
          case ApiKeys.ALTER_CONFIGS => handleAlterConfigsRequest(request)
          case ApiKeys.DESCRIBE_CONFIGS => handleDescribeConfigsRequest(request)
          case ApiKeys.ALTER_REPLICA_LOG_DIRS => handleAlterReplicaLogDirsRequest(request)
          case ApiKeys.DESCRIBE_LOG_DIRS => handleDescribeLogDirsRequest(request)
          case ApiKeys.SASL_AUTHENTICATE => handleSaslAuthenticateRequest(request)
          case ApiKeys.CREATE_PARTITIONS => handleCreatePartitionsRequest(request)
          case ApiKeys.CREATE_DELEGATION_TOKEN => handleCreateTokenRequest(request)
          case ApiKeys.RENEW_DELEGATION_TOKEN => handleRenewTokenRequest(request)
          case ApiKeys.EXPIRE_DELEGATION_TOKEN => handleExpireTokenRequest(request)
          case ApiKeys.DESCRIBE_DELEGATION_TOKEN => handleDescribeTokensRequest(request)
          case ApiKeys.DELETE_GROUPS => handleDeleteGroupsRequest(request)
        }
      } catch {
        case e: FatalExitError => throw e
        case e: Throwable => handleError(request, e)
      } finally {
        request.apiLocalCompleteTimeNanos = time.nanoseconds
      }
    }
  ```

  由🐎可知：

  * request有request.header.apiKey
  * 这个key在2.0.0版本中有43种命令

  