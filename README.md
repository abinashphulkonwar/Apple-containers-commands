# Start container system
container system start

# Stop container system
container system stop

# List all named volumes
container volume ls

# Remove specific named volumes
container volume rm app_node_modules_bpr npm_global_cache

# List all images
container image ls

# Remove the specific development image
container image rm blog-post-react

# System prune (Removes all stopped containers, unused networks, and dangling images)
container system prune

# System prune including all unused volumes
container system prune --volumes

# Build command with Docker file
container build -t imagename -f Dockerfile.dev .

# Volume
container run --rm -p 3000:3000 -v $(pwd):/app app_node_modules:/app/node_modules imagename

