# Specify the AWS provider
provider "aws" {
  region = "eu-north-1"  # Change to the region you want to use
}

# Launch Template
resource "aws_launch_template" "as_template" {
  name_prefix   = "web_config"
  image_id      = "ami-05389b1760f9d4a6e"  # Ensure this AMI is correct for your region
  instance_type = "t2.micro"
  key_name      = "prd"  # Ensure the key pair exists in the region

  

  # Define network interfaces to specify security groups
  network_interfaces {
    associate_public_ip_address = true  # Optional, if you want to associate a public IP
    security_groups            = ["sg-00b9bd4ea667c5ed3"]  # Security group as a list
  }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "asg" {
  desired_capacity        = 2
  max_size                = 5
  min_size                = 1
  vpc_zone_identifier     = ["subnet-0e3ed7a2749b7496f", "subnet-0859629926c2f38d1"]  # Subnet IDs
  launch_template {
    id      = aws_launch_template.as_template.id  # Reference to Launch Template ID
    version = "$Latest"  # Use the latest version of the Launch Template
  }
  health_check_type       = "EC2"  # Or "ELB" depending on your architecture
  health_check_grace_period = 300
  force_delete            = true  # Ensure that instances can be forcibly terminated when scaling down
  wait_for_capacity_timeout = "0"  # Avoid waiting for instance to be in "running" state before scaling

  # Tags for instances launched in the Auto Scaling group
  tag {
    key                 = "Name"
    value               = "AutoScalingInstance"
    propagate_at_launch = true
  }
}

# Scale-Up Policy
resource "aws_autoscaling_policy" "scale_up" {
  name                   = "scale_up_policy"
  scaling_adjustment     = 4                     # Increase by 4 instances
  adjustment_type        = "ChangeInCapacity"     # Fixed adjustment
  cooldown               = 300                   # 5 minutes cooldown
  autoscaling_group_name = aws_autoscaling_group.asg.id
}

# Scale-Down Policy
resource "aws_autoscaling_policy" "scale_down" {
  name                   = "scale_down_policy"
  scaling_adjustment     = -4                    # Decrease by 4 instances
  adjustment_type        = "ChangeInCapacity"     # Fixed adjustment
  cooldown               = 300                   # 5 minutes cooldown
  autoscaling_group_name = aws_autoscaling_group.asg.id
}


