package main

import (  "context","flag","fmt","log,"time"}

}

// Configuration for the cost optimization tool
type Config struct {
 Region string
 BucketName string
 ReportFile string
}

// Function to get AWS configuration
func getAWSConfig(region string) (*aws.Config, error) {
 cfg, err := config.LoadDefaultConfig(context.TODO(), config.WithRegion(region))
 if err != nil {
  return nil, err
 }
 return &cfg, nil
}

// Function to get EC2 instance pricing information
func getEC2Pricing(cfg *aws.Config, instanceType string) (float64, error) {
 // Create a Pricing client
 pricingClient := pricing.NewFromConfig(*cfg)

 // Define the pricing request parameters
 params := &pricing.GetProductsInput{
  ServiceCode:  aws.String("AmazonEC2"),
  Filters: []types.Filter{
   {
    Type:  aws.String("TERM_MATCH"),
    Field: aws.String("instanceType"),
    Value: []string{instanceType},
   },
  },
 }

 // Get pricing information
 response, err := pricingClient.GetProducts(context.TODO(), params)
 if err != nil {
  return 0, err
 }

 // Extract the price from the response
 if len(response.PriceList) > 0 {
  return parseFloat(response.PriceList[0].Price.Amount), nil
 }
 return 0, fmt.Errorf("price not found for instance type: %s", instanceType)
}

// Function to get S3 storage cost
func getS3StorageCost(cfg *aws.Config, bucketName string, storageClass string) (float64, error) {
 // Create an S3 client
 s3Client := s3.NewFromConfig(*cfg)

 // Define the pricing request parameters
 params := &pricing.GetProductsInput{
  ServiceCode:  aws.String("AmazonS3"),
  Filters: []types.Filter{
   {
    Type:  aws.String("TERM_MATCH"),
    Field: aws.String("storageClass"),
    Value: []string{storageClass},
   },
  },
 }

 // Get pricing information
 response, err := pricingClient.GetProducts(context.TODO(), params)
 if err != nil {
  return 0, err
 }

 // Extract the price from the response
 if len(response.PriceList) > 0 {
  return parseFloat(response.PriceList[0].Price.Amount), nil
 }
 return 0, fmt.Errorf("price not found for storage class: %s", storageClass)
}

// Function to get EC2 instances in a given region
func getEC2Instances(cfg *aws.Config, region string) ([]*ec2.Instance, error) {
 // Create an EC2 client
 ec2Client := ec2.NewFromConfig(*cfg)


 // Define the EC2 instance listing parameters
 params := &ec2.DescribeInstancesInput{
  Filters: []types.Filter{
   {
    Name:   aws.String("instance-state-name"),
    Values: []string{"running"},
   },
  },
 }

 // Get EC2 instance information
 response, err := ec2Client.DescribeInstances(context.TODO(), params)
 if err != nil {
  return nil, err
 }

 // Extract EC2 instances from the response
 var instances []*ec2.Instance
 for _, reservation := range response.Reservations {
  instances = append(instances, reservation.Instances...)
 }
 return instances, nil
}

// Function to write cost optimization report to a file
func writeReport(reportFile string, recommendations []string) error {
 // Create a file
 file, err := os.Create(reportFile)
 if err != nil {
  return err
 }
 defer file.Close()

 // Write recommendations to the file
 for _, recommendation := range recommendations {
  _, err = file.WriteString(recommendation + "n")
  if err != nil {
   return err
  }
 }
 return nil
}

// Function to parse a string to a float64
func parseFloat(str string) float64 {
 f, err := strconv.ParseFloat(str, 64)
 if err != nil {
  return 0
 }
 return f
}

func main() {
 // Parse command-line flags
 region := flag.String("region", "us-east-1", "AWS region")
 bucketName := flag.String("bucket-name", "", "S3 bucket name (optional)")
 reportFile := flag.String("report-file", "cost_optimization_report.txt", "Output report file")
 flag.Parse()

 // Get AWS configuration
 cfg, err := getAWSConfig(*region)
 if err != nil {
  log.Fatal("Error getting AWS configuration:", err)
 }

 // Configuration for the cost optimization tool
 config := Config{
  Region:     *region,
  BucketName: *bucketName,
  ReportFile: *reportFile,
 }

 // Get EC2 instances
 instances, err := getEC2Instances(cfg, config.Region)
 if err != nil {
  log.Println("Error getting EC2 instances:", err)
  return
 }

 // Get S3 storage information (optional)
 var s3StorageCost float64
 if config.BucketName != "" {
  s3StorageCost, err = getS3StorageCost(cfg, config.BucketName, "STANDARD")
  if err != nil {
   log.Println("Error getting S3 storage cost:", err)
  }
 }

 // Generate cost optimization recommendations
 var recommendations []string
 for _, instance := range instances {
  // Get EC2 instance pricing
  instancePrice, err := getEC2Pricing(cfg, *instance.InstanceType)
  if err != nil {
   log.Println("Error getting EC2 pricing:", err)
   continue
  }

  // Check if instance type can be downsized
  if instancePrice > 0 {
   recommendations = append(recommendations, fmt.Sprintf("Consider downsizing instance %s to a smaller instance type. Current price: $%.2f", *instance.InstanceId, instancePrice))
  }
 }

 // Add S3 storage cost recommendations (optional)
 if config.BucketName != "" && s3StorageCost > 0 {
  recommendations = append(recommendations, fmt.Sprintf("Consider using a different S3 storage class for bucket %s to reduce costs. Current cost: $%.2f", config.BucketName, s3StorageCost))
 }

 // Write cost optimization report to a file
 err = writeReport(config.ReportFile, recommendations)
 if err != nil {
  log.Fatal("Error writing report:", err)
 }

 fmt.Println("Cost optimization report generated successfully:", config.ReportFile)
}
