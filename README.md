# Overview
P&IDs are a specific type of widely used schematic diagram that show the logical connections for process facilities. They show the equipment, valves, pipes, and instruments and how they are connected. They do not convey scale, distance, oritentation or any spacial information. This project is take a P&ID in PDF form and convert it into data. It uses LLMs and more traditional ML techniques where LLMs currently don't do well. 
This project also prioritizes local operations where possible to minimize the need to use external resources. As of right now, the project relies on Claude Sonnet for some processing.

<img width="400" height="400" alt="test3_snip" src="https://github.com/user-attachments/assets/20232f94-07c2-4b39-81da-bacee5132ea1" />

## Problem Definition
The customer problem that I want to solve is what I call the handover problem, but it’s applicable in other scenarios. Imagine a process plant owner hires an engineering company to design and build a modification to their plant.

As part of their work, the engineering company creates a schematic diagram (P&ID) using their own tools and data structures. It’s all completely digital. 

The plant operators use completely different systems for operating the plant that don’t interoperate with the engineering companies data. Due to contractual, technical, and organizational factors, the engineering company typically turns over the data as images - CAD (DWG, DGN, DXF) or PDFs, etc. It hasn’t been that long since it was paper drawings.

The owner then needs to incorporate data from these drawings into their systems for operating and maintaining the plant.  There is a tremendous amount of data incorporated into the drawing, but it’s in human readable form and not readily consumed by computers. Some companies, such as Cognite have the ability to get some of the data from P&IDs. Last I saw, though it was a limited set of data.

Various people and standards bodies have tried to solve this problem for many, many years. The best that I’ve come across is DEXPI.org. They have defined a practical XML format and some of the CAD vendors have implemented imports or exports. Unfortunately, there is seldom an end to end solution for the plant owner.

My project is to take an arbitrary P&ID in PDF or some CAD format and, using AI/ML, create a DEXPI format XML file. Using XML is a convenient intermediate solution. Ultimately, I’d want to create the data directly into a target system. I can forsee using an agent to recreate the P&ID in the target system natively using the information pulled from the PDF. 

# Solution Details
## Hardware & OS
This solution was developed and tested on an Alienware Aurora R16 Intel Core i7-1400Fx28 with 64GB of memory and a NVIDIA GeForce RTX 4060 Ti graphics card. OS is Ubuntu 24.04.3.
## Algorithm
This solution is broken down into multiple steps
1. Convert the PDF into 4 pngs, upper left, upper right, lower left, lower right. We need high resolution images and at this resolution a single png would be too big for the tools to handle.
2. Get the drawing metadata. Using the bottom right png, read the title block data using Gemma3:27B. Prompt is *List the project name, company name, drawing (DWG) name, drawing number, location, date, revision, author, 
and any other information found in the title block in the lower right corner of the image. Return data as JSON*
3. Get the line list. To get the line list, you need all four quadrants. None of the local LLMs are able to handle more than one png at a time (though this problem is likely solveable). However none of the local LLMs do a credible job of retrieving a line list. This is likely to change as more capable models are released. Claude Sonnet does both - takes up to 4 files, and reliably extracts line list information. The cost per P&ID in my testing was less than $0.10 *list the pipe lines on this P&ID with the line number, primary size, and what equipment, off-page connectors, 
or other lines it is connected to. Return the data as JSON and only return JSON*
4. Get the equipment list. Local LLMs did not reliably return this information either. As with line list, Claude Sonnet was used. *List the equipment on this P&ID with the name, description and associated metadata. Only list the primary equipment. Do not list piping, valves, instruments or information about the drawing itself. Return data as JSON*
5. Valve List. LLMs don't do a good job at identifying all valve symbols. A ML algorithm, Yolov5, was used instead. This relies on work done previously here: https://github.com/ch-hristov/p-id-symbols. The training symbols are a generic set. If this is used commericially, you might want to retrain it using custom symbols that match the ones used on your P&IDs.
   5.1 Run each quadrant through Yolo to detect valves, instruments, other symbols. The result is a CSV file containing the class of object, confidence percent, and coordinates of the symbol in the drawing.
   5.2 Go through the results CSV file and create an png snippet centered around each symbol.
   5.3 Go through the CSV file again. Take each valve (and ignore the non- valve symbols) and send the png snippet to Gemma3:27B with the following prompt *Look at the valve in the center of the image. Return the valve size and valve tag number as a json string. If either value can't be found return the value as none*
6. Instrument list. This uses the same code as valves with a different list of classes and a different scaling factor. Prompt: *Look at the instrument in the center of the image. The instrument tag type is the top text
                    and the instrument number is the bottom text. Return a JSON string of top and bottom text with the keys top and bottom*
7. **TODO** Using the data gathered so far, create a DEXPI format XML file.

## Future work
 - See if I can train any of the LLM models to do better
 - Are there additional options for local processing so I replace Claude Sonnet with a local LLM

## Why priorizitize Local operations
### Data Privacy/Security

In my experience, many, if not most, operating companies regard their P&IDs as their intellectual property. The P&IDs contain details of how a plant turns raw materials into finished products and they want to keep this information confidential. This may or may not be a realistic view, but I’ve found it is a common one.

Secondly, many companies view their facilities as critical infrastructure. There are regulations that control how information about critical infrastructure can be disseminated. This doesn’t mean they can’t put P&IDs in cloud services, but it is a potential barrier.

Considering both of these, and the fact that the main LLM providers are known to store query data make potential customers for this system wary of uploading P&IDs to cloud services. It’s not an impossible barrier to overcome, but it is a barrier.

### Cost

The other factor is cost. There are two aspects of costs to consider. First, it’s the cost to develop the system. I’m doing this on my own without financial backing, so I want to minimize costs. The cloud costs could be substantial and unpredictable. I prefer using local resources where I have visibility into the compute and storage costs and can keep within a budget. Some of the cloud LLM services have fixed monthly cost accounts, but the amount of work these do is subject to change, which leads me to my second point on costs.

All the LLM services that I’ve seen so far define their costs based on the number of tokens used. Since the LLM service providers control both the generation of tokens and how they are counted, they have the ability to adjust prices and performance per dollar. While they are currently operating at a loss, as a business, they naturally will need to become profitable. It’s unclear if, how, and when this will occur. https://news.ycombinator.com/item?id=44567857

While the underlying tech gives amazing results, it’s unclear how much it’s fueled by burning investor money. One concern is that in a drive to become profitable, the costs to operate a cloud service will skyrocket and/or the capabilities will be neutered. I can avoid this risk while understanding the true costs by running my own local services 
