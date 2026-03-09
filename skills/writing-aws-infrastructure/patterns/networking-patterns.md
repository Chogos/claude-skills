# Networking Patterns

## Contents

- [Three-tier VPC layout](#three-tier-vpc-layout)
- [Security group chain: ALB → ECS → RDS](#security-group-chain-alb--ecs--rds)
- [VPC endpoint configurations](#vpc-endpoint-configurations)
- [Transit gateway for multi-VPC](#transit-gateway-for-multi-vpc)
- [NACLs for subnet-level filtering](#nacls-for-subnet-level-filtering)

## Three-tier VPC layout

Public subnets hold ALBs and NAT Gateways. Private subnets hold compute (ECS, Lambda, EC2). Isolated subnets hold databases — no route to the internet at all.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Three-tier VPC across 3 AZs

Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]
  VpcCidr:
    Type: String
    Default: "10.0.0.0/16"

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-vpc"

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-igw"

  IGWAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  # --- Public subnets (ALB, NAT GW) ---
  PublicSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: "10.0.0.0/20"
      AvailabilityZone: !Select [0, !GetAZs ""]
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-public-a"

  PublicSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: "10.0.16.0/20"
      AvailabilityZone: !Select [1, !GetAZs ""]
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-public-b"

  PublicSubnetC:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: "10.0.32.0/20"
      AvailabilityZone: !Select [2, !GetAZs ""]
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-public-c"

  # --- Private subnets (ECS, Lambda, EC2) ---
  PrivateSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: "10.0.48.0/20"
      AvailabilityZone: !Select [0, !GetAZs ""]
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-private-a"

  PrivateSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: "10.0.64.0/20"
      AvailabilityZone: !Select [1, !GetAZs ""]
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-private-b"

  PrivateSubnetC:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: "10.0.80.0/20"
      AvailabilityZone: !Select [2, !GetAZs ""]
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-private-c"

  # --- Isolated subnets (RDS, ElastiCache) ---
  IsolatedSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: "10.0.96.0/20"
      AvailabilityZone: !Select [0, !GetAZs ""]
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-isolated-a"

  IsolatedSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: "10.0.112.0/20"
      AvailabilityZone: !Select [1, !GetAZs ""]
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-isolated-b"

  IsolatedSubnetC:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: "10.0.128.0/20"
      AvailabilityZone: !Select [2, !GetAZs ""]
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-isolated-c"

  # --- NAT Gateways (one per AZ for prod HA) ---
  NatEipA:
    Type: AWS::EC2::EIP
    Properties:
      Domain: vpc

  NatGatewayA:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatEipA.AllocationId
      SubnetId: !Ref PublicSubnetA
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-nat-a"

  NatEipB:
    Type: AWS::EC2::EIP
    Properties:
      Domain: vpc

  NatGatewayB:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatEipB.AllocationId
      SubnetId: !Ref PublicSubnetB
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-nat-b"

  NatEipC:
    Type: AWS::EC2::EIP
    Properties:
      Domain: vpc

  NatGatewayC:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatEipC.AllocationId
      SubnetId: !Ref PublicSubnetC
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-nat-c"

  # --- Route tables ---
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-public-rt"

  PublicDefaultRoute:
    Type: AWS::EC2::Route
    DependsOn: IGWAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: "0.0.0.0/0"
      GatewayId: !Ref InternetGateway

  PublicSubnetARouteAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetA
      RouteTableId: !Ref PublicRouteTable

  PublicSubnetBRouteAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetB
      RouteTableId: !Ref PublicRouteTable

  PublicSubnetCRouteAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetC
      RouteTableId: !Ref PublicRouteTable

  PrivateRouteTableA:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-private-rt-a"

  PrivateDefaultRouteA:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTableA
      DestinationCidrBlock: "0.0.0.0/0"
      NatGatewayId: !Ref NatGatewayA

  PrivateSubnetARouteAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnetA
      RouteTableId: !Ref PrivateRouteTableA

  PrivateRouteTableB:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-private-rt-b"

  PrivateDefaultRouteB:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTableB
      DestinationCidrBlock: "0.0.0.0/0"
      NatGatewayId: !Ref NatGatewayB

  PrivateSubnetBRouteAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnetB
      RouteTableId: !Ref PrivateRouteTableB

  PrivateRouteTableC:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-private-rt-c"

  PrivateDefaultRouteC:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTableC
      DestinationCidrBlock: "0.0.0.0/0"
      NatGatewayId: !Ref NatGatewayC

  PrivateSubnetCRouteAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnetC
      RouteTableId: !Ref PrivateRouteTableC

  # Isolated subnets: no route to internet, no NAT
  IsolatedRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-isolated-rt"

  IsolatedSubnetARouteAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref IsolatedSubnetA
      RouteTableId: !Ref IsolatedRouteTable

  IsolatedSubnetBRouteAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref IsolatedSubnetB
      RouteTableId: !Ref IsolatedRouteTable

  IsolatedSubnetCRouteAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref IsolatedSubnetC
      RouteTableId: !Ref IsolatedRouteTable

Outputs:
  VpcId:
    Value: !Ref VPC
    Export:
      Name: !Sub "${Environment}-vpc-id"
  PublicSubnets:
    Value: !Join [",", [!Ref PublicSubnetA, !Ref PublicSubnetB, !Ref PublicSubnetC]]
  PrivateSubnets:
    Value: !Join [",", [!Ref PrivateSubnetA, !Ref PrivateSubnetB, !Ref PrivateSubnetC]]
  IsolatedSubnets:
    Value: !Join [",", [!Ref IsolatedSubnetA, !Ref IsolatedSubnetB, !Ref IsolatedSubnetC]]
```

**CIDR plan summary:**

| Tier | Subnet | CIDR | IPs |
|------|--------|------|-----|
| Public | a / b / c | 10.0.0.0/20 / 10.0.16.0/20 / 10.0.32.0/20 | 4091 each |
| Private | a / b / c | 10.0.48.0/20 / 10.0.64.0/20 / 10.0.80.0/20 | 4091 each |
| Isolated | a / b / c | 10.0.96.0/20 / 10.0.112.0/20 / 10.0.128.0/20 | 4091 each |
| Reserved | — | 10.0.144.0/20 – 10.0.240.0/20 | 7 blocks for growth |

## Security group chain: ALB → ECS → RDS

Each tier references the previous tier's security group ID. No CIDR-based rules for internal traffic.

```yaml
Resources:
  AlbSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: ALB - public HTTPS ingress
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: "0.0.0.0/0"
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-alb-sg"

  EcsSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: ECS tasks - ingress from ALB only
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 8080
          ToPort: 8080
          SourceSecurityGroupId: !Ref AlbSecurityGroup
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-ecs-sg"

  RdsSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: RDS - ingress from ECS only
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 5432
          ToPort: 5432
          SourceSecurityGroupId: !Ref EcsSecurityGroup
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-rds-sg"

  # Redis/ElastiCache behind ECS as well
  RedisSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Redis - ingress from ECS only
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 6379
          ToPort: 6379
          SourceSecurityGroupId: !Ref EcsSecurityGroup
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-redis-sg"
```

Traffic flow: Internet → ALB (443) → ECS (8080) → RDS (5432). Each hop is gated by security group reference.

## VPC endpoint configurations

Gateway endpoints are free and added to route tables. Interface endpoints create ENIs in your subnets and cost ~$0.01/hr per AZ + data processing.

```yaml
Resources:
  # --- Gateway endpoints (free) ---
  S3Endpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VPC
      ServiceName: !Sub "com.amazonaws.${AWS::Region}.s3"
      VpcEndpointType: Gateway
      RouteTableIds:
        - !Ref PrivateRouteTableA
        - !Ref PrivateRouteTableB
        - !Ref PrivateRouteTableC

  DynamoDBEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VPC
      ServiceName: !Sub "com.amazonaws.${AWS::Region}.dynamodb"
      VpcEndpointType: Gateway
      RouteTableIds:
        - !Ref PrivateRouteTableA
        - !Ref PrivateRouteTableB
        - !Ref PrivateRouteTableC

  # --- Interface endpoints (paid) ---
  EndpointSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: VPC interface endpoints
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: !Ref VpcCidr

  EcrApiEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VPC
      ServiceName: !Sub "com.amazonaws.${AWS::Region}.ecr.api"
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      SubnetIds:
        - !Ref PrivateSubnetA
        - !Ref PrivateSubnetB
        - !Ref PrivateSubnetC
      SecurityGroupIds:
        - !Ref EndpointSecurityGroup

  EcrDkrEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VPC
      ServiceName: !Sub "com.amazonaws.${AWS::Region}.ecr.dkr"
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      SubnetIds:
        - !Ref PrivateSubnetA
        - !Ref PrivateSubnetB
        - !Ref PrivateSubnetC
      SecurityGroupIds:
        - !Ref EndpointSecurityGroup

  SecretsManagerEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VPC
      ServiceName: !Sub "com.amazonaws.${AWS::Region}.secretsmanager"
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      SubnetIds:
        - !Ref PrivateSubnetA
        - !Ref PrivateSubnetB
        - !Ref PrivateSubnetC
      SecurityGroupIds:
        - !Ref EndpointSecurityGroup

  CloudWatchLogsEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VPC
      ServiceName: !Sub "com.amazonaws.${AWS::Region}.logs"
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      SubnetIds:
        - !Ref PrivateSubnetA
        - !Ref PrivateSubnetB
        - !Ref PrivateSubnetC
      SecurityGroupIds:
        - !Ref EndpointSecurityGroup
```

For ECS/Fargate pulling from ECR, you need all three: `ecr.api`, `ecr.dkr`, and the S3 gateway endpoint (ECR stores layers in S3).

## Transit gateway for multi-VPC

Central hub for routing between VPCs, on-premises, and shared services. Replaces VPC peering mesh.

```yaml
Resources:
  TransitGateway:
    Type: AWS::EC2::TransitGateway
    Properties:
      Description: Central hub
      AutoAcceptSharedAttachments: disable
      DefaultRouteTableAssociation: enable
      DefaultRouteTablePropagation: enable
      DnsSupport: enable
      VpnEcmpSupport: enable
      Tags:
        - Key: Name
          Value: central-tgw

  ProdAttachment:
    Type: AWS::EC2::TransitGatewayAttachment
    Properties:
      TransitGatewayId: !Ref TransitGateway
      VpcId: !Ref ProdVPC
      SubnetIds:
        - !Ref ProdPrivateSubnetA
        - !Ref ProdPrivateSubnetB
        - !Ref ProdPrivateSubnetC
      Tags:
        - Key: Name
          Value: prod-attachment

  SharedServicesAttachment:
    Type: AWS::EC2::TransitGatewayAttachment
    Properties:
      TransitGatewayId: !Ref TransitGateway
      VpcId: !Ref SharedServicesVPC
      SubnetIds:
        - !Ref SharedPrivateSubnetA
        - !Ref SharedPrivateSubnetB
        - !Ref SharedPrivateSubnetC
      Tags:
        - Key: Name
          Value: shared-services-attachment
```

- Attach one subnet per AZ — the TGW routes to all subnets in that AZ.
- Use separate route tables for isolation (prod cannot route to dev).
- $0.05/hr per attachment + $0.02/GB data processing.

## NACLs for subnet-level filtering

NACLs are stateless — you must define both inbound and outbound rules. Use them as a coarse defense layer; security groups handle fine-grained control.

```yaml
Resources:
  IsolatedNACL:
    Type: AWS::EC2::NetworkAcl
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-isolated-nacl"

  # Allow inbound from private subnets on database ports only
  IsolatedInboundPostgres:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref IsolatedNACL
      RuleNumber: 100
      Protocol: 6  # TCP
      RuleAction: allow
      CidrBlock: "10.0.48.0/18"  # all private subnets
      PortRange:
        From: 5432
        To: 5432

  # Allow return traffic on ephemeral ports
  IsolatedInboundEphemeral:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref IsolatedNACL
      RuleNumber: 200
      Protocol: 6
      RuleAction: allow
      CidrBlock: "10.0.48.0/18"
      PortRange:
        From: 1024
        To: 65535

  # Deny all other inbound
  IsolatedInboundDenyAll:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref IsolatedNACL
      RuleNumber: 32767
      Protocol: -1
      RuleAction: deny
      CidrBlock: "0.0.0.0/0"

  # Allow outbound responses to private subnets
  IsolatedOutboundEphemeral:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref IsolatedNACL
      RuleNumber: 100
      Protocol: 6
      Egress: true
      RuleAction: allow
      CidrBlock: "10.0.48.0/18"
      PortRange:
        From: 1024
        To: 65535

  # Deny all other outbound — isolated subnets have no internet
  IsolatedOutboundDenyAll:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref IsolatedNACL
      RuleNumber: 32767
      Protocol: -1
      Egress: true
      RuleAction: deny
      CidrBlock: "0.0.0.0/0"

  IsolatedSubnetANACLAssoc:
    Type: AWS::EC2::SubnetNetworkAclAssociation
    Properties:
      SubnetId: !Ref IsolatedSubnetA
      NetworkAclId: !Ref IsolatedNACL

  IsolatedSubnetBNACLAssoc:
    Type: AWS::EC2::SubnetNetworkAclAssociation
    Properties:
      SubnetId: !Ref IsolatedSubnetB
      NetworkAclId: !Ref IsolatedNACL

  IsolatedSubnetCNACLAssoc:
    Type: AWS::EC2::SubnetNetworkAclAssociation
    Properties:
      SubnetId: !Ref IsolatedSubnetC
      NetworkAclId: !Ref IsolatedNACL
```

- Rules are evaluated in order by RuleNumber — lower numbers first.
- Ephemeral port range (1024–65535) must be open for return traffic because NACLs are stateless.
- Keep NACLs simple — overly complex NACLs are hard to debug and often conflict with security groups.