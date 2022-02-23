#!/bin/bash

# Define colours
red=$'\e[1;31m'
grn=$'\e[1;32m'
yel=$'\e[1;33m'
blu=$'\e[1;34m'
mag=$'\e[1;35m'
cyn=$'\e[1;36m'
end=$'\e[0m'
diskAttached="false"
defectiveDisk=""
message=""

#Print banner and version of the script
echo "############################################################################"
echo "${mag}SAS DRIVE TESTING SCRIPT                                         VERSION 1.4${end}"
echo "############################################################################"

cd /DiskTests

# Check if disk is attached
while [ "$diskAttached" = "false" ];
	do
	 	if [ "$(sudo smartctl --scan | grep $1 | wc -l)" -lt "1" ]; then
			echo
			echo "${yel}Test #1 FAIL: Disk is not detected. Trying again in two seconds.${end}"
			echo
			sleep 2
		fi
	 	if [ "$(sudo smartctl --scan | grep $1 | wc -l)" -gt "0" ]; then
			echo "${mag}INFO: Found disk $1${end}"
			((diskAttached="true"))
		fi
	done

#Check if the disk transport protocol is SATA 
if [[ $(hdparm -I /dev/$1 | grep "Transport" | grep "SATA" | wc -l) -eq 0 || $(hdparm -I /dev/$1 | grep "Transport" | grep "Serial" | wc -l) -eq 0 || $(hdparm -I /dev/$1 | grep "ATA device" | wc -l) -eq 0 ]]; then
	#statements
	echo "${cyn}SAS disk detected${end}"
	echo ""
else
	while [[ true ]]; do
		clear
		echo "${yel}This is not a SAS disk${end}"
		sleep 3
	done
fi

#Erase old SMART report file
if [ -e "smartOutput_$1" ]; then
	rm smartOutput_$1
fi

#Check if the disk is 512 block size
if [ "$(sudo blockdev --getsz /dev/$1)" -eq "0" ]; then
	echo
	echo "${red}ERROR: This disk needs to be reformatted to 512 block size first before testing.${end}"
fi

if [ "$(sudo blockdev --getsz /dev/$1)" -gt "0" ]; then
	echo "${mag}INFO: Disk is 512 block size${end}"

	# Generate a report and echo to file "smartOutput_$1"
	sudo smartctl -T permissive -x /dev/$1 > smartOutput_$1

	#Print SMART data to the user
	cat smartOutput_$1 | grep "Vendor"
	cat smartOutput_$1 | grep "Product"
	cat smartOutput_$1 | grep "Capacity"
	cat smartOutput_$1 | grep "Serial"
	cat smartOutput_$1 | grep "Revision"

	#Check for imminent failure
	if [[ $(cat smartOutput_$1 | grep "FAIL" | wc -l) -eq 1 ]]; then
		echo "${red}_________________________________________________________________${end}"
		echo ""
		echo "${red}HEALTH                     : WARNING DISK FAILURE IS IMMINENT !!!${end}"
		echo "${red}_________________________________________________________________${end}"
		echo ""
		defectiveDisk="true"
	else
		healthAssesment=$(cat smartOutput_$1 | grep "SMART Health" | awk '{print $NF}')
	fi

	# If grep does not find "hours powered" try to grep "power on time"
	if [[ $(cat smartOutput_$1 | grep "hours powered" | awk '{print $NF}' | wc -l) -lt 1 ]]; then
		# If grep does not find "power on time" set powerOnHours value to an error
		if [[ $(cat smartOutput_$1 | grep "power on time" | awk '{print $NF}' | wc -l) -lt 1 ]]; then
			powerOnHours='Error! Unable to get power on hours. Please ask for help.'
			powerOnHoursToInt='error'			
		fi
		if [[ $(cat smartOutput_$1 | grep "power on time" | awk '{print $NF}' | wc -l) -gt 0 ]]; then
			powerOnHours=$(cat smartOutput_$1 | grep "power on time" | awk '{print $6}' | sed 's/[:].*$//')
			powerOnHoursToInt=$powerOnHours			
		fi
	fi
	
	# If power on hours are reported set the powerOnHours variable to that value
	if [[ $(cat smartOutput_$1 | grep "hours powered" | awk '{print $NF}' | wc -l) -gt 0 ]]; then
		powerOnHours=$(cat smartOutput_$1 | grep "hours powered" | awk '{print $NF}')
		#Convert decimal value of $powerOnHours to integer
    	powerOnHoursToInt=$( printf "%.0f" $powerOnHours )
	fi	
        
    #Power on hours value
    # < 5 days
	if [[ $powerOnHoursToInt -lt 120 ]]; then
	  hourValue="${grn}New Drive 0 Hours${end}"
	fi

	# < 30 Days: 
	if [[ $powerOnHoursToInt -gt 119 && $powerOnHoursToInt -lt 720 ]]; then
	  hourValue="${blu}New Pull < 30 Days${end}"
	fi

	# < 60 Days:
	if [[ $powerOnHoursToInt -gt 719 && $powerOnHoursToInt -lt 1440 ]]; then
	  hourValue="${org}New Pull < 60 Days${end}"
	fi

	# < 3 Years:
	if [[ $powerOnHoursToInt -gt 1439 && $powerOnHoursToInt -lt 26280 ]]; then
	  hourValue="${mag}RFB < 3 Years${end}"
	fi

	# > 3 Years:
	if [[ $powerOnHoursToInt -gt 26279 ]]; then
	  hourValue="${cyn}USED > 3 Years${end}"
	fi

	# No hours reported - error
	if [[ $powerOnHoursToInt == "error" ]]; then
	  hourValue="Unable to set HOUR VALUE!"
	fi
	
	#Make disk grade represent a color
	grownDefects=$(cat smartOutput_$1 | grep 'grown defect' | awk '{print $NF}')
	#A Grade
	if [[ $grownDefects -lt  1 ]]; then
		grade="${grn}A${end}"
	fi
	
	#B Grade
	if [[ $grownDefects -lt 50 && $grownDefects -gt 0 ]]; then
		grade="${cyn}B${end}"
	fi
	
	#C Grade
	if [[ $grownDefects -gt 49 ]]; then
  		grade="${org}C (FAIL)${end}"
	fi
    
	sudo rm smartOutput_$1
	echo "############################################################################"
	echo "                            ${mag}DISK TESTING PHASE${end}                              "
	echo "############################################################################"
	echo "${mag}INFO: Checking disk protection type${end}"
	if [[ $(sg_readcap -l /dev/$1 | grep "Protection" | awk '{print $2}') == "prot_en=0," && $(sg_readcap -l /dev/$1 | grep "Protection" | awk '{print $3}') == "p_type=0," ]]; then
		echo "${mag}OK. This disk is FreeNAS compatible.${end}"
	else
		while [[ true ]]; do
			#statements
			clear
			echo "${red}ERROR: This SAS disk is NOT formatted with Type 0 Protection! (Not FreeNAS compatible)${end}"
			echo "${yel}To fix this issue, please run a 512 block size conversion on this disk first and test again.${end}"
			defectiveDisk="true"
			message=$(echo "${yel}This disk is NOT defective but will likely be RMA'd. Please run a 512 block size conversion on this disk first before shipping / placing into inventory.${end}")
			sleep 3
		done
	fi
	echo "${mag}INFO: Unmounting disk and partitions${end}"
	sudo umount /dev/$1*
    if [ $? != 0 ]; then
    	echo "${mag}Disk is not mounted${end}"
    else
    	echo "${mag}Disk was unmounted${end}"
    fi	
	echo "${mag}INFO: Removing RAID lock and existing partitions${end}"
	sudo wipefs -a /dev/$1
	if [ $? != 0 ]; then
    	echo "${red}ERROR REMOVING RAID LOCK AND OR PARTITIONS${end}"
    else
    	echo "${mag}OK${end}"
	fi
	echo "${mag}INFO: Creating a EXT4 file system${end}"
	# sudo mkfs.vfat -I -F 32 -n HDD /dev/$1
	sudo mkfs.ext4 -F /dev/$1
	#Check if creating a file system passed or not (partition test)
	if [ $? != 0 ]; then
		echo "${red}ERROR CREATING A EXT4 FILE SYSTEM${end}"
		defectiveDisk="true"
	else
		echo "${mag}OK${end}"
	fi

	# Remove ext4 filesystem
	echo "${mag}INFO: Removing ext4 file system${end}"
	sudo wipefs -a /dev/$1
	sudo dd if=/dev/zero of=/dev/$1 bs=1M count=100
	if [ $? != 0 ]; then
		echo "${red}ERROR REMOVING EXT4 FILE SYSTEM${end}"
		defectiveDisk="true"
	else
		echo "${mag}OK${end}"
	fi

	# Show disk test results
	if [[ $defectiveDisk == "true" ]];then
		echo "############################################################################" 
		# Change the background color to red
		echo -e "\e[48;5;9m"
		echo "                                                                            "
		echo "                                                                            "
		echo "                                  RED LABEL                                 "
		echo "                                                                            "
		echo "                                                                            "
		echo -e "\e[0m"

		# Print SMART information
		echo $message
		echo "DISK TEST RESULT : ${red}DEFECTIVE DISK${end}"
		echo ""		
		echo "POWER ON HOURS   : $powerOnHours"
		echo "GROWN DEFECTS    : $grownDefects"
		echo "GRADE            : ${red}FAIL${end}"
		echo ""
		echo "############################################################################"
	else
		echo "############################################################################" 
					
		# Change the color of the background depending on the disk grade

		# A Grade
		if [[ $grade == "${grn}A${end}" ]]; then
			echo -e "\e[48;5;2m"
			echo "                                                                            "
			echo "                                                                            "
			echo "                              GRADE A LABEL                                 "
			echo "                                                                            "
			echo "                                                                            "
			echo -e "\e[0m"
		fi

		# B Grade
		if [[ $grade == "${cyn}B${end}" ]]; then
			echo -e "\e[48;5;27m"
			echo "                                                                            "
			echo "                                                                            "
			echo "                              GRADE B LABEL                                 "
			echo "                                                                            "
			echo "                                                                            "
			echo -e "\e[0m"
		fi

		# C Grade
		if [[ $grade == "${org}C (FAIL)${end}" ]]; then
			echo -e "\e[48;5;208m"
			echo "                                                                            "
			echo "                                                                            "
			echo "                              GRADE C LABEL                                 "
			echo "                                                                            "
			echo "                                                                            "
			echo -e "\e[0m"
		fi

		# Print SMART information
		echo "DISK TEST RESULT : PASSED"
		echo "HEALTH ASSESSMENT: $healthAssesment"
		echo "POWER ON HOURS   : $powerOnHours"
		echo "GROWN DEFECTS    : $grownDefects"
		echo "GRADE            : $grade"
		echo "HOUR VALUE       : $hourValue"
		echo ""
		echo "############################################################################" 
	fi
	
	# Re-read partition table
	sudo partprobe /dev/$1

	echo -n
	read response
fi
read input
