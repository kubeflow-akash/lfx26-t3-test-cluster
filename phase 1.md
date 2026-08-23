TEST_OCID=ocid1.cluster.oc1.iad.aaaaaaaaejqldnhqdjkzd366aey7prmf65i7hk75cnhwuchwpcpr53ata63a

oci ce cluster create-kubeconfig \                                                                                                                                                                                                                                                                            ─╯
  --cluster-id "$TEST_OCID" \
  --file ~/.kube/config \
  --region us-ashburn-1 \
  --token-version 2.0.0 \
  --auth security_token