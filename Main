--!native
--!optimize 2
local RenderLib = {};

RenderLib.mainColor = Color3.new(0.501961, 0.74902, 0.901961);

RenderLib.Scene = {};
RenderLib.Lights = {};
RenderLib.PixelTable = nil;

type Sphere = {
	Origin: Vector3;
	Radius: number;
	Color: Color3;
	Reflectance: number;
	TypeName: string;
}

type Box = {
	CFrame: CFrame;
	Size: Vector3;
	Color: Color3;
	Reflectance: number;
	TypeName: string;
}

type Wedge = {
	CFrame: CFrame;
	Size: Vector3;
	Color: Color3;
	Reflectance: number;
	TypeName: string;
}

type Cylinder = {
	CFrame: CFrame;
	Height: number;
	Radius: number;
	Color: Color3;
	Reflectance: number;
	TypeName: string;
}

type Plane = {
	Origin: Vector3;
	Normal: Vector3;
	Color: Color3;
	Reflectance: number;
	TypeName: string;
}

type PointLight = {
	Origin: Vector3;
	Color: Color3;
	Range: number;
	TypeName: string;
}

type RaycastResult2 = {
	Distance: number;
	NumSteps: number;
	NumBounces: number;
	Position: Vector3;
	Normal: Vector3;
	Color: Color3;
	TypeName: RaycastResult2
}

function RenderLib:InsertScene(object: Sphere | Box | Plane)
	table.insert(self.Scene, object);
end

function RenderLib:InsertLight(object: PointLight)
	table.insert(self.Lights, object);
end

function RenderLib:InitializePixelTable(totalPixels: number)
	self.PixelTable = table.create(totalPixels);
end

function RenderLib:InsertPixelTable(chunkTable:{Color3}, chunkSize:number, startPos:number)
	table.move(chunkTable, 1, chunkSize, startPos, self.PixelTable);
end

function RenderLib:WriteBuffer(pixelsBuffer): buffer	
	for i, pixel in self.PixelTable do
		local j = 4 * (i - 1);
		buffer.writeu8(pixelsBuffer, j + 0, math.floor(pixel.R * 255));
		buffer.writeu8(pixelsBuffer, j + 1, math.floor(pixel.G * 255));
		buffer.writeu8(pixelsBuffer, j + 2, math.floor(pixel.B * 255));
		buffer.writeu8(pixelsBuffer, j + 3, 255);
	end
	
	return pixelsBuffer;
end

--Quadratic smoothmin
function RenderLib.smoothMin(a, b, intensity): number
	local h = 1 - math.min(math.abs(a-b)/ (4 * intensity), 1);
	local w = h * h;
	local m = w * 0.5;
	local s = w * intensity;
	if (a < b) then
		return a - s, m;
	else
		return b - s, 1 - m;
	end
end

function RenderLib.linearStep(x:number): number
	return x * 0.5 + 0.5;
end

function RenderLib.smoothStep(x:number): number
	return x * x * (3 - 2 * x);
end

--Fractal brownian motion
function RenderLib.FBM(x:number, y:number, z:number)
	local h = 0.8;
	local numOctaves = 7;
	
	local g = 2^(-h);
	local f = 1;
	local a = 1;
	local sum = 0;
	for i = 1, numOctaves do
		sum += a * math.noise(f * x, f * y, f * z);
		f *= 2;
		a *= g;
	end
	
	return sum;
end


function RenderLib.newSphere(origin:Vector3, radius:number, color:Color3, reflectance:number): Sphere
	local object: Sphere = {};
	object.Origin = origin;
	object.Radius = radius;
	object.Color = color;
	object.Reflectance = reflectance;
	object.TypeName = "Sphere";
	
	return object;
end

function RenderLib.newBox(cframe:CFrame, size:Vector3, color:Color3, reflectance:number): Box
	local object: Box = {};
	object.CFrame = cframe;
	object.Size = size;
	object.Color = color;
	object.Reflectance = reflectance;
	object.TypeName = "Box";
	
	return object;
end

function RenderLib.newWedge(cframe:CFrame, size:Vector3, color:Color3, reflectance:number): Wedge
	local object: Wedge = {};
	object.CFrame = cframe;
	object.Size = size;
	object.Color = color;
	object.Reflectance = reflectance;
	object.TypeName = "Wedge";
	
	return object;
end

function RenderLib.newCylinder(cframe:CFrame, size:Vector3, color:Color3, reflectance:number): Cylinder
	local object: Cylinder = {};
	object.CFrame = cframe;
	object.Height = size.X;
	object.Radius = math.min(size.Y, size.Z);
	object.Color = color;
	object.Reflectance = reflectance;
	object.TypeName = "Cylinder";
	
	return object;
end

function RenderLib.newPlane(origin:Vector3, normal:Vector3, color:Color3, reflectance:number): Plane
	local object: Plane = {};
	object.Origin = origin;
	object.Normal = normal;
	object.Color = color;
	object.Reflectance = reflectance;
	object.TypeName = "Plane";

	return object;
end

function RenderLib.newLight(origin:Vector3, color:Color3, range:number, reflectance:number): PointLight
	local object: PointLight = {};
	object.Origin = origin;
	object.Color = color;
	object.Range = range;
	object.Reflectance = reflectance;
	object.TypeName = "PointLight";
	
	return object;
end

function RenderLib.distToSphere(point:Vector3, sphere:Sphere): number
	return (point - sphere.Origin).Magnitude - sphere.Radius;
end

function RenderLib.distToBox(point:Vector3, box:Box): number
	point = box.CFrame:PointToObjectSpace(point);
	point = point:Abs() - box.Size * 0.5;
	return point:Max(Vector3.zero).Magnitude + math.min(math.max(point.X, math.max(point.Y, point.Z)), 0);
end

--truly horrid
function RenderLib.distToWedge(point:Vector3, wedge:Wedge): number
	local halfWidth = wedge.Size.X * 0.5;
	local halfHeight = wedge.Size.Y * 0.5;
	local halfDepth = wedge.Size.Z * 0.5;
	local pointXY = Vector2.new(point.X, point.Y);
	point = wedge.CFrame:PointToObjectSpace(point);
	
	local e = Vector2.new(halfWidth, -halfHeight);
	local d1 = Vector2.new(-halfWidth - point.X, 
		math.max(math.abs(point.Y) - halfHeight, 0));
	local d2 = pointXY - e * math.clamp(pointXY:Dot(e) / e:Dot(e), 
		-1, 1);
	local d3 = Vector2.new(math.max(math.abs(point.X) - halfWidth, 0), 
		-halfHeight - point.Y);
	local d4 = math.sqrt(math.min(math.min(d1:Dot(d1), 
		d2:Dot(d2), d3:Dot(d3))) * math.sign(math.max(math.max(math.max(d1.X, d2.X), d2.Y), d3.Y)))
	local d5 = math.abs(point.Z) - halfDepth;
	return Vector2.new(d4, d5):Max(Vector2.zero).Magnitude + math.min(math.max(d4, d5), 0);
end

function RenderLib.distToCylinder(point:Vector3, cylinder:Cylinder): number
	point = cylinder.CFrame:PointToObjectSpace(point);
	
	local d = Vector2.new(Vector2.new(point.X, point.Z).Magnitude, point.Y):Abs() 
		- Vector2.new(cylinder.Radius, cylinder.Height);
	return math.min(math.max(d.X, d.Y), 0) + (d:Max(Vector2.zero)).Magnitude;
end

function RenderLib.distToPlane(point:Vector3, plane:Plane): number
	return (point - plane.Origin):Dot(plane.Normal);
end

local distHandler = {
	Sphere = RenderLib.distToSphere;
	Box = RenderLib.distToBox;
	Wedge = RenderLib.distToWedge;
	Cylinder = RenderLib.distToCylinder;
	Plane = RenderLib.distToPlane;
}

Color3.new(0.2, 0.5, 0.6)

--some cool dist functions are listed below:
--[[OCEAN:
local returnColor = Color3.new(0.2, 0.5, 0.5 + (math.sin((point.Y - 10)/5) * 0.1 + 0.1))--Color3.new(math.sin(point.Magnitude) * 0.5 + 0.5, math.sin(os.clock()) * 0.5 + 0.5, math.cos(point.Magnitude) * 0.5 + 0.5);
	local minDist = RenderLib.distToPlane(
		point + Vector3.new(0,2,0) * 
			math.noise(
				point.X / 10, 
				math.sin(os.clock()), 
				point.Z / 10 + os.clock()
				), 
		RenderLib.Scene[2]);
]]


function RenderLib.distToScene(point:Vector3): number | Color3 | nil
	local scene = RenderLib.Scene;
	local sceneLen = #scene;
	if sceneLen == 0 then return nil end

	local minDist = distHandler[scene[1].TypeName](point, scene[1]);
	local returnColor = scene[1].Color;
	local returnReflectance = scene[1].Reflectance;
	
	for i = 2, sceneLen do
		local object = scene[i];
		local dist = distHandler[object.TypeName](point, object);
		
		if dist < minDist then
			minDist = dist;
			returnColor = object.Color;
			returnReflectance = object.Reflectance;
		end
	end
	return minDist, returnColor, returnReflectance;
end

function RenderLib.calcNormal(point:Vector3): Vector3
	local epsilon = 0.01;
	local dx = Vector3.new(epsilon,0,0);
	local dy = Vector3.new(0,epsilon,0);
	local dz = Vector3.new(0,0,epsilon);
	
	local nx = RenderLib.distToScene(point + dx) - RenderLib.distToScene(point - dx);
	local ny = RenderLib.distToScene(point + dy) - RenderLib.distToScene(point - dx);
	local nz = RenderLib.distToScene(point + dz) - RenderLib.distToScene(point - dx);
	return Vector3.new(nx, ny, nz).Unit;
end

function RenderLib.raycast(rayOrigin:Vector3, rayDirection:Vector3): RaycastResult2 | nil
	local epsilon = 0.01;
	
	local maxSteps = 1024;
	local maxBounces = 0;
	local maxDist = 100;
	
	local numBounces = 0;
	local totalDist = 0;
	local oldDist = 0;
	
	local raycastResult: RaycastResult2 = {};
	raycastResult.TypeName = "RaycastResult2";
	
	for step = 1, maxSteps do
		local distance, baseColor, reflectance = RenderLib.distToScene(rayOrigin + rayDirection * totalDist);
		--empty scene, no objects to render
		if distance == nil then break end;
		
		--check if object hit
		if distance < epsilon then
			local hitPos = rayOrigin + rayDirection * (oldDist);
			local n = Vector3.zero --RenderLib.calcNormal(hitPos);
			if numBounces < maxBounces and reflectance > 0 then
				rayOrigin = hitPos;
				rayDirection = rayDirection - 2 * (rayDirection:Dot(n)) * n;
				totalDist = 0;
				numBounces += 1;
			else
				raycastResult.Position = hitPos;
				raycastResult.NumSteps = step;
				raycastResult.Distance = totalDist;
				raycastResult.Normal = n;
				raycastResult.Color = baseColor;
				return raycastResult;
			end
		end
		
		if (totalDist > maxDist) then break end
		
		oldDist = totalDist;
		totalDist += math.abs(distance);
	end
	
	return nil;
end

--modified verion of raycast for soft shadows
function RenderLib.castShadow(rayOrigin:Vector3, rayDirection:Vector3, softness:number): number
	local maxSteps = 1024;
	
	local maxDist = 100;
	local totalDist = 0;
	local result = 1;
	
	local lastResult;
	
	for i = 1, maxSteps do
		local distance = RenderLib.distToScene(rayOrigin + rayDirection * totalDist);
		result = math.min(result, distance / (softness * totalDist));
		totalDist += math.clamp(distance, 0.05, 0.5);
		
		if (result < -1 or totalDist > maxDist) then break end
		
		totalDist += math.abs(distance);
	end
	result = math.max(result, -1);
	return 0.25 * (1 + result) * (1 + result) * (2 - result);
end

function RenderLib.renderPixel(pixelPosition:Vector2, cameraCF:CFrame, size:Vector2)
	local scene = RenderLib.Scene;
	local lights = RenderLib.Lights;
	local pixelColor = RenderLib.mainColor;
	local horizontalCoefficient = pixelPosition.X/size.X - 0.5;
	local verticalCoefficient = 0.5 - pixelPosition.Y/size.Y;

	local rayOrigin = cameraCF.Position;
	local rayDirection = (cameraCF.LookVector + cameraCF.UpVector * verticalCoefficient + cameraCF.RightVector * horizontalCoefficient).Unit;

	local result = RenderLib.raycast(rayOrigin, rayDirection, scene);

	if result then
		local lightVector = (lights[1].Origin - result.Position).Unit;
		local brightness = RenderLib.castShadow(result.Position, lightVector, 0.2);
		pixelColor = result.Color
			--:Lerp(Color3.new(0,0,0), math.clamp(result.Normal:Dot(Vector3.new(-1,-1,0)), 0, 1))
			--:Lerp(Color3.new(1,1,1), math.clamp(result.NumSteps, 0, 256)/512) --glow
			--:Lerp(Color3.new(0.878431, 0.890196, 0.247059), 0.5 - 0.5 * result.Normal:Dot(Vector3.new(0,1,0)))--normals
			:Lerp(Color3.new(0, 0, 0), 0.9 - 0.5 * brightness) --shadows
	end
	
	return pixelColor;
end

--stores in pixelTable
function RenderLib.renderChunk(size:Vector2, cameraCF:CFrame, numActors:number, actorIndex:number)
	local chunkSizeX = size.X;
	local chunkSizeY = math.ceil(size.Y/numActors);
	local startY = (actorIndex - 1) * chunkSizeY;
	chunkSizeY = math.min(chunkSizeY, size.Y - startY);
	
	local startPos = startY * chunkSizeX + 1;
	local chunkSize = chunkSizeX * chunkSizeY;
	
	local chunkTable = table.create(chunkSize);
	local pixelPosition;
	
	for y = startY, startY + chunkSizeY - 1 do
		for x = 0, chunkSizeX - 1 do
			pixelPosition = Vector2.new(x, y);
			table.insert(
				chunkTable, 
				RenderLib.renderPixel(pixelPosition, cameraCF, size)
			);			
		end
	end

	return chunkTable, chunkSize, startPos;
end

return RenderLib;

--test1 - 113 ms, 50x50
--test2 - 107ms, decreased maxSteps to 64. not worth it
--test3 - increasing epsilon to 0.1 - 84ms, increasing epsilon to 1 - 65 ms. could be worthwhile?
--test4 - native boosts to 80ms. APPLIED >T10
--test5 - ignoring normal calculations boosts to 100ms. normal isn't used, so could be worth it. APPLIED >T10
--test6 - 2249ms, 200x200
--test7 - replaced divisions with multiplications, 2000ms. KEEPING
--test8 - implemented distHander. 1750ms -> 1710ms. KEEPING
--test9 - distToScene outputs two values instead of a table. 1558ms. KEEPING
--test10 - test9 optimizations applied to smoothMin. 1283ms. KEEPING
--test11 - native & ignore normals. 800ms. KEEPING
--test12 - disabling smoothmin calculations. 454 ms. might keep?
--test13 - removed camera.cframe calls in renderPixel. 390ms. KEEPING
--test14 - added parallelization. 205ms with 3 threadss. KEEPING
--test15 - increased clamp min from 0.005 to 0.05 in shadowcast. 193ms KEEPING
